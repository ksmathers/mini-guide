---
title: PySpark in EKS
description: 
published: true
date: 2025-12-06T05:26:21.271Z
tags: 
editor: markdown
dateCreated: 2025-12-06T05:11:03.613Z
---


 ✅ Guide: Running Spark on EKS from Jupyter with S3 Access (Manual Role Annotation)

## 1. Architecture
- **Goal:** Submit Spark jobs from Jupyter to an EKS cluster, reading/writing Parquet data in S3.
- **Control:** Jupyter notebook uses PySpark to talk directly to the Kubernetes API.

---

## 2. Spark on Kubernetes
- Spark uses the **Kubernetes API** to create driver and executor pods.
- Your Jupyter notebook acts as the **client**.
- Spark authenticates using your **kubeconfig** (same as `kubectl`), which supports AWS IAM Auth via SSO.

---

## 3. Authentication & IAM
- No AWS keys required.
- You will **manually annotate pods** with the IAM role ARN so they can access S3.
- Spark supports **pod templates** for driver and executor pods, where you can add annotations.

---

## 4. Networking
- If `kubectl` works, Spark can use the same kubeconfig.
- The spark session builder will start driver and executor pods by using a reference to the kubernetes cluster control node.
- A code snippet setting the EKS API endpoint from your ~/.kube/config would look like this:
  ```python
  ...
  spark = SparkSession.builder \
    .appName("RemoteSparkJob") \
    .master("k8s://https://<EKS_API_ENDPOINT>") \
  ...
  ```

---

## 5. Spark Image
Build a custom Spark image with S3 support:
```dockerfile
FROM bitnami/spark:latest
ADD hadoop-aws-3.x.jar /opt/spark/jars/
ADD aws-java-sdk-bundle-1.x.jar /opt/spark/jars/
```
Push to a registry accessible by EKS.  Then add a reference to that 
image name to the session builder:
```python
...
    .config("spark.kubernetes.container.image", "<your-spark-image>") \
...
```

---

## 6. Pod Templates for IAM Role
Since you cannot create ServiceAccounts, use pod templates:

**Driver Template (`driver-template.yaml`):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    spark-role: driver
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<role-name>
spec:
  containers:
    - name: spark-kubernetes-driver
      image: <your-spark-image>
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "2"
          memory: "4Gi"
```

**Executor Template (`executor-template.yaml`):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    spark-role: executor
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<role-name>
spec:
  containers:
    - name: spark-kubernetes-executor
      image: <your-spark-image>
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "2"
          memory: "4Gi"
```

Adding these to Spark session builder:
```python
...
  .config("spark.kubernetes.driver.podTemplateFile", "/path/to/driver-template.yaml") \
  .config("spark.kubernetes.executor.podTemplateFile", "/path/to/executor-template.yaml") \
...
```

---

## 7. PySpark Session in Jupyter

Condense all of the preceding configuration steps into your Jupyter notebook PySpark initialization step.  

For example:
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("RemoteSparkJob") \
    .master("k8s://https://<EKS_API_ENDPOINT>") \
    .config("spark.kubernetes.container.image", "<your-spark-image>") \
    .config("spark.kubernetes.namespace", "spark") \
    .config("spark.kubernetes.executor.instances", "4") \
    .config("spark.kubernetes.driver.podTemplateFile", "/path/to/driver-template.yaml") \
    .config("spark.kubernetes.executor.podTemplateFile", "/path/to/executor-template.yaml") \
    .getOrCreate()
```

---

## 8. S3 Access
- Use `s3a://bucket-name/path` for input/output.
- AWS SDK automatically picks up credentials from the IAM role assigned to the pod.
- Example:
```python
df = spark.read.parquet("s3a://my-bucket/input/")
df.write.mode("overwrite").parquet("s3a://my-bucket/output/")
```

---

### ✅ Key Points
- No hardcoded AWS credentials.
- No need for IRSA or new ServiceAccounts—annotations in pod templates handle IAM.
- Direct Kubernetes API access via kubeconfig and AWS SSO.
