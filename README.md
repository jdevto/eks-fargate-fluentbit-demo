# eks-fargate-fluentbit-demo

## Fluent Bit for Amazon EKS on AWS Fargate Setup

This guide provides clear steps to configure Fluent Bit for logging in an Amazon EKS cluster running on AWS Fargate.

### Prerequisites

- An existing EKS cluster
- AWS CLI configured with necessary permissions
- `kubectl` set up to access your EKS cluster

---

## Step 1: Deploy the EKS Cluster

Before setting up Fluent Bit, create an EKS cluster with Fargate support using `eksctl`.

Create a file called `eks-cluster-config.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fluentbit-cluster
  region: ap-southeast-1
  version: "1.32"

fargateProfiles:
  - name: default
    selectors:
      - namespace: demo
      - namespace: aws-observability
```

Then deploy the cluster:

```bash
eksctl create cluster --config-file eks-cluster-config.yaml
```

> ⚠️ This may take several minutes. Once complete, `kubectl` will automatically be configured to use the new cluster.

---

### Step 2: Create Fluent Bit Configuration

Create a file called `fluentbit-config.yaml` with the following content:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: aws-observability
  labels:
    aws-observability: enabled
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-logging
  namespace: aws-observability
  labels:
    app.kubernetes.io/name: fluent-bit
    k8s-app: fluent-bit
    component: logging
    tier: node
    release: fluentbit
  annotations:
    fluentbit.io/exclude: "false"
data:
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match   *
        region ap-southeast-1
        log_group_name fluent-bit-cloudwatch
        log_stream_prefix from-fluent-bit-
        auto_create_group true
```

> ⚠️ **Important:** Be sure to use `true` (lowercase) for `auto_create_group`, or the configuration will be rejected by Fargate’s admission webhook.

Apply the configuration:

```bash
kubectl apply -f fluentbit-config.yaml
```

Confirm the ConfigMap was created:

```bash
kubectl -n aws-observability get configmap aws-logging -o yaml
```

---

### Step 3: Set Up IAM Permissions

Download the required IAM policy:

```bash
curl -o permissions.json \
     https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json
```

Create the IAM policy:

```bash
aws iam create-policy \
        --policy-name FluentBitEKSFargate \
        --policy-document file://permissions.json
```

Attach the policy to your Fargate pod execution role:

```bash
aws iam attach-role-policy \
        --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/FluentBitEKSFargate \
        --role-name eksctl-fluentbit-cluster-FargatePodExecutionRole-XXXXXXXXXX
```

Replace:

- `<YOUR_ACCOUNT_ID>` with your AWS account ID
- `eksctl-fluentbit-cluster-FargatePodExecutionRole-XXXXXXXXXX` with your actual Fargate pod execution role

---

### Step 4: Deploy Test Application

Create a file called `logger-server.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logger-server
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: main
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Deploy the test application:

```bash
kubectl create namespace demo
```

Creates a separate namespace for your test workload.

```bash
kubectl -n demo apply -f logger-server.yaml
```

Applies the deployment configuration.

```bash
kubectl -n demo expose deploy logger-server
```

Exposes the deployment as a service within the `demo` namespace.

---

### Step 5: Verify Logging

> 📌 **Make sure Fluent Bit is deployed and running in `aws-observability` before proceeding:**

```bash
kubectl get pods -n aws-observability
```

You should see a `fluent-bit` pod with `STATUS` as `Running`.

#### 1. Forward the service locally

> ⚠️ **If the pod is still `Pending`, wait before forwarding.**

```bash
kubectl get pods -n demo
```

Then:

```bash
kubectl -n demo port-forward svc/logger-server 8080:80
```

> 💡 Tip: Open a new terminal session for other steps while this command is running.

#### 2. Stream logs

```bash
kubectl -n demo logs deploy/logger-server -f
```

> 💡 Tip: Use another terminal session for log streaming while port-forwarding is active.

#### 3. Send requests

```bash
curl localhost:8080
```

---

### Step 6: Check CloudWatch Logs

Logs are available in CloudWatch under the group:

```plaintext
fluent-bit-cloudwatch
```

Check available log groups:

```bash
aws logs describe-log-groups --region ap-southeast-1
```

Example output:

```json
{
  "logGroups": [
    {
      "logGroupName": "/aws/eks/fluentbit/cluster",
      "creationTime": 1746219458104,
      "arn": "arn:aws:logs:ap-southeast-1:xxxxxxxxxx:log-group:/aws/eks/fluentbit/cluster:*"
    },
    {
      "logGroupName": "fluent-bit-cloudwatch",
      "creationTime": 1746222769503,
      "arn": "arn:aws:logs:ap-southeast-1:xxxxxxxxxx:log-group:fluent-bit-cloudwatch:*"
    }
  ]
}
```

List log streams:

```bash
aws logs describe-log-streams --log-group-name fluent-bit-cloudwatch --region ap-southeast-1
```

Example output:

```json
{
  "logStreams": [
    {
      "logStreamName": "from-fluent-bit-kube.var.log.containers.logger-server-...",
      "creationTime": 1746222769529,
      "firstEventTimestamp": 1746222769404
    }
  ]
}
```

---

### Resource Considerations

- Allocate 50MB–100MB additional memory for the Fargate log router.
- Consider CloudWatch ingestion and retention costs.

---

### Supported Kubernetes Versions

Tested on Kubernetes v1.32. May work on other versions as well.

---

### Cleanup

#### 1. Delete namespaces (this also deletes all pods inside automatically)

```bash
kubectl delete namespace demo
```

Removes the test application namespace.

```bash
kubectl delete namespace aws-observability
```

Removes Fluent Bit and its configuration.

#### 2. Detach and delete IAM policy

```bash
aws iam detach-role-policy \
  --role-name eksctl-fluentbit-cluster-FargatePodExecutionRole-XXXXXXXXXX \
  --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/FluentBitEKSFargate
```

Detaches the Fluent Bit policy.

```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/FluentBitEKSFargate
```

Deletes the policy itself.

#### 3. Delete the EKS cluster

```bash
eksctl delete cluster --config-file eks-cluster-config.yaml
```

Removes the entire EKS cluster created during setup.

---

### Reference

Based on: <https://aws.amazon.com/blogs/containers/fluent-bit-for-amazon-eks-on-aws-fargate-is-here/>
