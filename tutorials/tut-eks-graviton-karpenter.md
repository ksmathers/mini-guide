---
title: tut-eks-graviton-karpenter
description: 
published: true
date: 2026-04-27T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-04-27T00:00:00.000Z
---

# EKS with Dynamic Graviton Node Provisioning Using Karpenter

Modern Kubernetes clusters on AWS face a practical tension: you want nodes to appear when workloads need them and disappear when they do not, but you also want those nodes to be cost-effective and fast. This tutorial builds an EKS cluster that solves that problem — Karpenter watches for unschedulable pods and provisions ARM64 Graviton nodes (t4g, m7g, or m8g) on-demand within seconds, then consolidates them away when the work is done.

By the end of this tutorial you will understand:

- What Graviton is, why ARM64 matters for your container images, and how the t4g, m7g, and m8g families differ
- How Karpenter's provisioning loop works and how it differs from the older Cluster Autoscaler model
- How EKS Pod Identity binds Kubernetes service account identity to AWS IAM roles without static credentials or OIDC configuration
- How `NodePool` and `EC2NodeClass` work together to express what nodes Karpenter is allowed to launch
- How consolidation reclaims underutilized nodes automatically

> **Quick-start:** If you only need Karpenter deployed and running without the explanation, the upstream [getting started guide](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/) covers that in fewer steps.

---

## 📋 Overview

The end-state architecture has three layers:

| Layer | Component | Role |
|---|---|---|
| **Control plane** | EKS managed control plane | Kubernetes API server, etcd, scheduler |
| **System nodes** | Small managed node group (x86 or arm64) | Runs Karpenter itself and other system DaemonSets |
| **Workload nodes** | Karpenter-managed Graviton EC2 instances | Dynamically provisioned per-pod-demand |

Karpenter runs as a controller in the system node group. It watches the Kubernetes scheduler for pods that cannot be placed (status `Pending` due to insufficient resources) and calls the EC2 API directly to launch new nodes. When those pods leave, or when nodes are underutilized, Karpenter drains and terminates the nodes.

This is deliberately different from the Cluster Autoscaler (CA) model, which is covered in [Part 1](#-part-1-graviton-instance-types-and-the-arm64-contract) and again in [Part 4](#-part-4-installing-karpenter).

---

## ⚙️ Prerequisites and Configuration Variables

**Required tools:**

- `aws` CLI ≥ 2.x, configured with a profile that has `AdministratorAccess` or a scoped IAM policy covering EKS, EC2, IAM, and SQS
- `eksctl` ≥ 0.195 (install via Homebrew: `brew tap weaveworks/tap && brew install weaveworks/tap/eksctl`)
- `kubectl` ≥ 1.29
- `helm` ≥ 3.14

Verify all four are present before proceeding:

```bash
aws --version && eksctl version && kubectl version --client && helm version --short
```

**Configuration variables** — set these once; every command block below references them:

```bash
# Your AWS account ID
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# The region to deploy into
export AWS_REGION="us-east-1"

# A name for your cluster — used as a tag value for subnet/SG discovery
export CLUSTER_NAME="graviton-demo"

# Kubernetes version to use
export K8S_VERSION="1.32"

# Karpenter version — check https://github.com/aws/karpenter-provider-aws/releases
export KARPENTER_VERSION="1.12.0"

# Verify all are set
echo "Account: $AWS_ACCOUNT_ID | Region: $AWS_REGION | Cluster: $CLUSTER_NAME"
```

> ⚠️ **Warning:** All resources created in this tutorial incur AWS costs. The EKS control plane alone costs ~$0.10/hr. Run `eksctl delete cluster --name $CLUSTER_NAME` when you are done to avoid charges.

---

## 🔧 Part 1: Graviton Instance Types and the ARM64 Contract

### What Graviton is

AWS Graviton processors are ARM-based CPUs designed by AWS. They differ from Intel/AMD x86 CPUs in their instruction set architecture (ISA): ARM64 (also written `aarch64`) vs. `x86_64`. This matters because compiled binaries and container images are ISA-specific — an `x86_64` image cannot run on an ARM64 node without emulation.

The three families used in this tutorial are all ARM64:

| Family | Graviton Gen | Class | vCPU range | Use when |
|---|---|---|---|---|
| **t4g** | 2 | Burstable | 2–8 | Dev, CI, low/variable traffic workloads |
| **m7g** | 3 | General purpose | 1–64 | Steady-state application servers |
| **m8g** | 4 | General purpose | 2–96 | Latest generation; best perf/dollar for most apps |

**Burstable vs. general purpose** is the key distinction between t4g and the m-families. t4g instances accumulate CPU credits when idle and consume them during bursts. If a workload sustains high CPU for longer than the credit balance allows, the instance throttles. m7g and m8g provide full, unthrottled vCPU at all times. Use t4g for workloads with spiky or intermittent CPU profiles; use m7g/m8g for anything with sustained load.

> **Alternative:** If you have existing `x86_64`-only container images and cannot rebuild them, use `c7i`, `m7i`, or `m6i` instead of Graviton. Graviton is cost-optimal (typically 20–40% cheaper per vCPU than equivalent x86 instances) but requires ARM64 images.

### The ARM64 container image requirement

When Karpenter provisions a Graviton node and Kubernetes schedules a pod onto it, the pod's container image must support `linux/arm64`. You can check whether an image is multi-arch before deploying:

```bash
# Inspect manifest list — look for linux/arm64 in the platforms
docker buildx imagetools inspect public.ecr.aws/amazonlinux/amazonlinux:2023
```

Official images from Docker Hub, ECR Public, and most major projects ship multi-arch manifests. The Docker daemon automatically pulls the correct platform layer. Problems arise with internal images built only for `amd64`.

To build a multi-arch image from a standard Dockerfile:

```bash
# One-time: create a builder that can cross-compile
docker buildx create --name multi --use

# Build and push both architectures in a single step
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag your-registry/your-image:latest \
  --push .
```

> **Concept:** A multi-arch image is actually a *manifest list* — a JSON envelope that maps platform tuples (`os/arch`) to individual image digests. The Docker daemon and Kubernetes CRI (containerd) resolve the correct digest automatically when pulling.

### How Karpenter enforces the architecture contract

When Karpenter selects a node for a pending pod, it intersects the pod's node selector/affinity requirements with the NodePool's `requirements` field. Because you will set `kubernetes.io/arch: arm64` in the NodePool, Karpenter will only provision ARM64 nodes. If a pod has no architecture affinity and the image is multi-arch, this works correctly. If a pod has an explicit `nodeSelector: kubernetes.io/arch: amd64`, Karpenter will not schedule it on this NodePool — the pod will stay Pending until a compatible NodePool or node exists.

---

## 🔧 Part 2: Creating the EKS Cluster

### Why you still need a small system node group

Karpenter runs as a Kubernetes controller (a Deployment). Before Karpenter exists, there are no nodes for it to run on. You need at least one node group that is *not* managed by Karpenter to host Karpenter itself, CoreDNS, the VPC CNI, and kube-proxy.

This node group is intentionally small — two nodes of a modest instance type. It is not where your application workloads run.

```bash
# Create the cluster with a minimal system node group
# eksctl writes a kubeconfig entry automatically
eksctl create cluster \
  --name "${CLUSTER_NAME}" \
  --version "${K8S_VERSION}" \
  --region "${AWS_REGION}" \
  --nodegroup-name system \
  --node-type m7g.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --managed \
  --tags "karpenter.sh/discovery=${CLUSTER_NAME}"
```

This takes approximately 15–20 minutes. eksctl is calling CloudFormation under the hood: it creates a VPC with public and private subnets, the EKS control plane, and the managed node group.

> **Alternative:** Terraform (`terraform-aws-modules/eks`) and AWS CDK (`aws-cdk-lib/aws-eks`) are common alternatives. They give you more control over VPC design, tagging, and GitOps integration. eksctl is faster for learning because it makes sensible defaults opinionated. For production, Terraform or CDK is preferred because infrastructure-as-code is auditable and repeatable.

### Verify the cluster

```bash
# Confirm nodes are Ready
kubectl get nodes -L kubernetes.io/arch,node.kubernetes.io/instance-type

# Confirm you can reach the API server
kubectl cluster-info
```

Expected output: two nodes with `ARCH=arm64` (because `m7g.medium` is Graviton) and `STATUS=Ready`.

### Tag subnets for Karpenter discovery

Karpenter discovers which subnets and security groups to use by looking for a specific tag. eksctl tagged the cluster-level resources with `karpenter.sh/discovery=${CLUSTER_NAME}` during creation, but the subnets need that tag too:

```bash
# Get the private subnet IDs created by eksctl
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=tag:alpha.eksctl.io/cluster-name,Values=${CLUSTER_NAME}" \
  --query "Subnets[].SubnetId" \
  --output text)

# Tag each subnet
for SUBNET_ID in $SUBNET_IDS; do
  aws ec2 create-tags \
    --resources "${SUBNET_ID}" \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
done

# Verify
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}" \
  --query "Subnets[].{ID:SubnetId,Zone:AvailabilityZone,CIDR:CidrBlock}" \
  --output table
```

> **Concept:** Tag-based discovery decouples the Karpenter configuration from specific resource IDs. If subnets are rotated or new AZs are added, Karpenter picks them up automatically as long as the tag is present. Hard-coding subnet IDs in `EC2NodeClass` is an alternative that gives you explicit control but requires manual updates when the infrastructure changes.

---

## 🔧 Part 3: IAM Roles for Karpenter

Karpenter needs two IAM roles:

| Role | Who assumes it | What it does |
|---|---|---|
| `KarpenterNodeRole` | EC2 instances (nodes) | Lets nodes call ECR, VPC CNI, SSM |
| `KarpenterControllerRole` | Karpenter controller pod (via Pod Identity) | Lets Karpenter call EC2, IAM, SQS APIs |

### What EKS Pod Identity is and why it matters

Traditionally, EC2 instances use instance profiles (attached IAM roles) to get AWS credentials. Every pod on that instance shares those credentials — if one pod is compromised, all AWS access on that node is exposed.

EKS Pod Identity solves this by binding a specific IAM role to a specific Kubernetes ServiceAccount in a specific namespace, without needing OIDC federation or ServiceAccount annotations. The flow is:

1. You install the `eks-pod-identity-agent` DaemonSet (an EKS managed addon) onto every node
2. You create a *Pod Identity Association* — an EKS API object that maps `namespace/service-account` → IAM role ARN
3. When a pod starts, the agent intercepts IMDS credential requests and returns short-lived credentials scoped to the associated role
4. Only pods running under the matched ServiceAccount in the matched namespace receive those credentials

The IAM role's trust policy simply trusts the `pods.eks.amazonaws.com` service principal — no cluster-specific OIDC URL, no `sub` condition to maintain.

> **Alternative:** IRSA (IAM Roles for Service Accounts) is the older mechanism and uses the EKS OIDC provider as a trust broker. It is still functional but is now deprecated in favor of Pod Identity. IRSA requires enabling an OIDC provider per cluster and annotating each ServiceAccount with the role ARN — more moving parts than Pod Identity. Prefer Pod Identity for all new clusters.

### Create the node role

```bash
# Create the node role — this is what EC2 instances assume
cat > /tmp/node-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file:///tmp/node-trust-policy.json

# Attach the three policies every EKS worker node needs
for POLICY in \
  AmazonEKSWorkerNodePolicy \
  AmazonEKS_CNI_Policy \
  AmazonEC2ContainerRegistryReadOnly \
  AmazonSSMManagedInstanceCore; do
  aws iam attach-role-policy \
    --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --policy-arn "arn:aws:iam::aws:policy/${POLICY}"
done

# Create an instance profile and link it to the role
aws iam create-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"

aws iam add-role-to-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}" \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}"
```

The `AmazonSSMManagedInstanceCore` policy is not strictly required for EKS but lets you open shell sessions to Karpenter-provisioned nodes via AWS Systems Manager Session Manager without needing SSH keys — useful for debugging.

### Register the node role with EKS

EKS uses a ConfigMap (`aws-auth`) to map IAM roles/users to Kubernetes RBAC subjects. Karpenter-provisioned nodes must be allowed to join the cluster:

```bash
# Add the node role to aws-auth so nodes can register with the cluster
eksctl create iamidentitymapping \
  --cluster "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --username "system:node:{{EC2PrivateDNSName}}" \
  --group "system:bootstrappers" \
  --group "system:nodes"
```

### Install the EKS Pod Identity agent

The Pod Identity agent is a DaemonSet that runs on every node and intercepts credential requests from pods. It is distributed as an EKS managed addon:

```bash
# Install the agent addon
aws eks create-addon \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --addon-name eks-pod-identity-agent

# Wait for it to become active before proceeding
aws eks wait addon-active \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --addon-name eks-pod-identity-agent

# Verify the agent DaemonSet is running on all nodes
kubectl get daemonset -n kube-system eks-pod-identity-agent
```

Expected: the DaemonSet shows `DESIRED` equal to the number of nodes and all pods `READY`.

### Create the controller role for Pod Identity

The trust policy for a Pod Identity role is simpler than IRSA — it trusts the `pods.eks.amazonaws.com` service principal with no cluster-specific OIDC URL or `sub` condition:

```bash
cat > /tmp/controller-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
EOF

aws iam create-role \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file:///tmp/controller-trust-policy.json
```

> **Concept:** `sts:TagSession` is required alongside `sts:AssumeRole` for Pod Identity. The EKS Pod Identity service uses session tags to pass context (cluster name, namespace, service account name, pod name) into the assumed-role session. These tags are available as condition keys in IAM policies if you want finer-grained access control downstream.

### Attach the Karpenter controller policy

Karpenter needs permission to describe and tag EC2 resources, launch and terminate instances, pass the node IAM role to instances, and optionally receive SQS messages for spot interruption notices. The upstream project maintains a managed policy document:

```bash
# Download the current policy from the Karpenter release
curl -o /tmp/karpenter-controller-policy.json \
  "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" 2>/dev/null || true

# Use the inline version below — this is equivalent to the managed policy at v1.12.x
cat > /tmp/karpenter-controller-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowScopedEC2InstanceActions",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:ec2:${AWS_REGION}::image/*",
        "arn:aws:ec2:${AWS_REGION}::snapshot/*",
        "arn:aws:ec2:${AWS_REGION}:*:spot-instances-request/*",
        "arn:aws:ec2:${AWS_REGION}:*:security-group/*",
        "arn:aws:ec2:${AWS_REGION}:*:subnet/*",
        "arn:aws:ec2:${AWS_REGION}:*:launch-template/*",
        "arn:aws:ec2:${AWS_REGION}:*:instance/*",
        "arn:aws:ec2:${AWS_REGION}:*:volume/*",
        "arn:aws:ec2:${AWS_REGION}:*:network-interface/*",
        "arn:aws:ec2:${AWS_REGION}:*:placement-group/*",
        "arn:aws:ec2:${AWS_REGION}:*:fleet/*"
      ],
      "Action": [
        "ec2:RunInstances",
        "ec2:CreateFleet",
        "ec2:CreateLaunchTemplate"
      ]
    },
    {
      "Sid": "AllowScopedEC2InstanceActionsWithTags",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:ec2:${AWS_REGION}:*:instance/*",
        "arn:aws:ec2:${AWS_REGION}:*:launch-template/*"
      ],
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteLaunchTemplate"
      ],
      "Condition": {
        "StringLike": {
          "aws:ResourceTag/karpenter.sh/nodepool": "*"
        }
      }
    },
    {
      "Sid": "AllowScopedResourceCreationTagging",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:ec2:${AWS_REGION}:*:instance/*",
        "arn:aws:ec2:${AWS_REGION}:*:launch-template/*",
        "arn:aws:ec2:${AWS_REGION}:*:volume/*",
        "arn:aws:ec2:${AWS_REGION}:*:network-interface/*"
      ],
      "Action": "ec2:CreateTags",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned"
        },
        "StringLike": {
          "aws:RequestTag/karpenter.sh/nodepool": "*"
        }
      }
    },
    {
      "Sid": "AllowMachineMigrationTagging",
      "Effect": "Allow",
      "Resource": "arn:aws:ec2:${AWS_REGION}:*:instance/*",
      "Action": "ec2:CreateTags",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
          "aws:RequestTag/karpenter.sh/nodeclaim": "migrated"
        }
      }
    },
    {
      "Sid": "AllowScopedDeletion",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:ec2:${AWS_REGION}:*:instance/*",
        "arn:aws:ec2:${AWS_REGION}:*:launch-template/*"
      ],
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteLaunchTemplate"
      ],
      "Condition": {
        "StringLike": {
          "aws:ResourceTag/karpenter.sh/nodepool": "*"
        }
      }
    },
    {
      "Sid": "AllowRegionalReadActions",
      "Effect": "Allow",
      "Resource": "*",
      "Action": [
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypeOfferings",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplates",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSpotPriceHistory",
        "ec2:DescribeSubnets"
      ]
    },
    {
      "Sid": "AllowSSMReadActions",
      "Effect": "Allow",
      "Resource": "arn:aws:ssm:${AWS_REGION}::parameter/aws/service/*",
      "Action": "ssm:GetParameter"
    },
    {
      "Sid": "AllowPassingInstanceRole",
      "Effect": "Allow",
      "Resource": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
      "Action": "iam:PassRole"
    },
    {
      "Sid": "AllowScopedInstanceProfileActions",
      "Effect": "Allow",
      "Resource": "*",
      "Action": [
        "iam:AddRoleToInstanceProfile",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:TagInstanceProfile"
      ]
    },
    {
      "Sid": "AllowInstanceProfileReadActions",
      "Effect": "Allow",
      "Resource": "*",
      "Action": "iam:GetInstanceProfile"
    },
    {
      "Sid": "AllowAPIServerEndpointDiscovery",
      "Effect": "Allow",
      "Resource": "arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}",
      "Action": "eks:DescribeCluster"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --policy-name "KarpenterControllerPolicy" \
  --policy-document file:///tmp/karpenter-controller-policy.json
```

The `Condition` blocks on the `TerminateInstances` and `DeleteLaunchTemplate` actions scope Karpenter's destructive powers to resources it created. This is a security boundary: Karpenter cannot terminate an EC2 instance it did not launch because the instance will not have the `karpenter.sh/nodepool` tag.

### Create the Pod Identity Association

The association is the binding that tells EKS: "when a pod in namespace `karpenter` running as ServiceAccount `karpenter` starts, inject credentials for this IAM role":

```bash
aws eks create-pod-identity-association \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --namespace karpenter \
  --service-account karpenter \
  --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}"

# Verify the association was created
aws eks list-pod-identity-associations \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --namespace karpenter
```

> **Concept:** The association lives in the EKS control plane, not in the cluster itself. You can list, update, and delete associations via the `aws eks` CLI without touching Kubernetes objects. This also means the binding survives cluster upgrades and namespace deletions — you must explicitly delete the association when decommissioning Karpenter.

---

## 🔧 Part 4: Installing Karpenter

### How Karpenter's provisioning loop works (and how it differs from Cluster Autoscaler)

**Cluster Autoscaler** monitors the size of Auto Scaling Groups (ASGs) and scales them up or down. It operates on pre-defined node groups — you must declare the instance types in advance, and the CA calls ASG `SetDesiredCapacity` to add nodes. Scaling is typically measured in minutes, and the CA does not optimize bin-packing (it will not move pods between nodes to consolidate).

**Karpenter** operates at the Kubernetes pod level, not the ASG level. When the scheduler marks a pod as `Pending` due to insufficient resources, Karpenter:

1. Reads the pod's resource requests, node selectors, and affinity rules
2. Chooses the cheapest instance type from the NodePool's allowed set that fits the pod (bin-packing)
3. Calls the EC2 `RunInstances` or `CreateFleet` API directly — no ASG involved
4. Waits for the node to register with the cluster and removes the `karpenter.sh/unregistered` taint
5. Kubernetes then schedules the pod onto the new node

This direct EC2 API approach means Karpenter can provision nodes in ~30–60 seconds. It can also select across dozens of instance types per decision, which gives it much more flexibility to find available capacity (important for Spot instances) and to choose the most cost-efficient size.

> **Alternative:** Use Cluster Autoscaler if you need predictable scaling behavior tied to fixed ASG configurations, have compliance requirements that mandate pre-approved instance types, or need to integrate with a capacity reservation workflow that Karpenter doesn't support. For most new EKS deployments, Karpenter is the recommended approach.

### Install via Helm

```bash
# Add the Karpenter Helm repository
helm repo add karpenter https://charts.karpenter.sh/
helm repo update

# Get the EKS cluster endpoint — Karpenter needs this to bootstrap nodes
CLUSTER_ENDPOINT=$(aws eks describe-cluster \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --query "cluster.endpoint" \
  --output text)

# Install Karpenter into its own namespace
# No ServiceAccount annotation needed — Pod Identity handles credential injection
helm upgrade --install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --version "${KARPENTER_VERSION}" \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.clusterEndpoint=${CLUSTER_ENDPOINT}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

With Pod Identity there is no ServiceAccount annotation to set. The Pod Identity agent on the node intercepts the AWS SDK credential call and provides credentials for the associated role automatically, based on the pod's namespace and ServiceAccount name.

`interruptionQueue` references an SQS queue that receives EC2 Spot interruption notices, EC2 instance rebalance recommendations, and health events. When Karpenter receives a two-minute Spot interruption warning, it proactively drains the node and reschedules its pods before AWS reclaims the instance. Creating the SQS queue and the EventBridge rules is out of scope for this tutorial but is strongly recommended for Spot workloads — see the [upstream guide](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/#create-the-karpenter-infrastructure-and-iam-roles).

### Verify Karpenter is running

```bash
kubectl get pods -n karpenter
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=20
```

Expected: two Karpenter pods with status `Running`. The logs should contain lines like `Starting controller` and `Watching for NodePool changes`.

---

## 🔧 Part 5: Defining the EC2NodeClass

An `EC2NodeClass` is a Karpenter CRD that captures the AWS-specific configuration for nodes: which AMI to use, which subnets and security groups to attach, which IAM role to assume, and how to configure the EBS root volume.

One `EC2NodeClass` can be referenced by multiple `NodePool` objects. Think of it as the "machine template" and `NodePool` as the "scheduling policy".

### AMI selection for Graviton

The `amiSelectorTerms.alias` field selects EKS-optimized AMIs maintained by AWS. The alias `al2023@latest` resolves to the current Amazon Linux 2023 image for your EKS version and the architecture of the instance being launched. Because Karpenter knows the instance's architecture before selecting the AMI, it automatically picks the `arm64` variant for Graviton nodes.

> **Concept:** AL2023 (Amazon Linux 2023) replaced AL2 as the default EKS AMI family at Kubernetes 1.29. AL2 reached end-of-support at Kubernetes 1.32. Use `al2023` unless you have a specific reason to use Bottlerocket (covered below).

> **Alternative:** Bottlerocket is an AWS-built container-optimized OS with an immutable root filesystem, automatic OS updates via A/B partition flipping, and a smaller attack surface. It is a good choice for security-sensitive production environments but has less tooling available for interactive debugging. Specify `alias: bottlerocket@latest` to use it.

```bash
# Get the node security group created by EKS for the cluster
CLUSTER_SG=$(aws eks describe-cluster \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
  --output text)

kubectl apply -f - <<EOF
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: graviton
spec:
  # al2023@latest: always use the newest AL2023 EKS-optimized AMI.
  # In production, pin this to a specific version (e.g. al2023@v20250410)
  # and validate new AMI versions in a lower environment before rolling forward.
  amiSelectorTerms:
    - alias: al2023@latest

  # IAM role for the nodes — created in Part 3
  role: "KarpenterNodeRole-${CLUSTER_NAME}"

  # Subnet discovery: any subnet tagged karpenter.sh/discovery=<cluster>
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"

  # Security group discovery: attach the cluster's shared node SG plus any
  # SG tagged for this cluster.
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
    - id: "${CLUSTER_SG}"

  # Root EBS volume: gp3 is faster and cheaper than gp2 for the same size.
  # 50Gi is enough for container image layers on nodes running a few workloads;
  # increase for nodes that pull many large images.
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
        deleteOnTermination: true

  # IMDSv2 only: prevents pods from accessing instance metadata without
  # going through the hop-limit, reducing the blast radius of SSRF attacks.
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 1
    httpTokens: required

  tags:
    ClusterName: "${CLUSTER_NAME}"
    ManagedBy: karpenter
EOF
```

### Verify EC2NodeClass readiness

```bash
kubectl get ec2nodeclass graviton -o wide
```

The `READY` column should be `True`. If it is `False`, describe the resource for a diagnostic message:

```bash
kubectl describe ec2nodeclass graviton
```

Common readiness failures: subnets not found (missing tag), security groups not found (missing tag), or instance profile not found (IAM not yet propagated — wait 30 seconds and retry).

---

## 🔧 Part 6: Defining the NodePool

A `NodePool` declares the scheduling constraints Karpenter must respect when choosing an instance type. It also controls disruption behavior — how aggressively Karpenter consolidates nodes when workloads shrink.

```bash
kubectl apply -f - <<EOF
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: graviton
spec:
  template:
    metadata:
      labels:
        # Custom label applied to every node in this pool.
        # Workloads can use nodeSelector to target Graviton nodes explicitly.
        eks-node-pool: graviton
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: graviton

      requirements:
        # ARM64 only — enforces Graviton instance selection and ensures
        # multi-arch images are required for all scheduled pods.
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]

        # The three Graviton families for this tutorial.
        # t4g: burstable, cost-effective for variable workloads.
        # m7g: Graviton 3, general purpose, steady-state.
        # m8g: Graviton 4, general purpose, latest generation.
        # Karpenter chooses between them per-pod based on resource fit and cost.
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["t4g", "m7g", "m8g"]

        # Exclude micro and nano sizes — too small to run typical workloads
        # alongside node-level DaemonSets (CNI, log forwarder, etc.)
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: ["nano", "micro"]

        # Prefer spot but fall back to on-demand if spot is unavailable.
        # Karpenter prioritizes: reserved > spot > on-demand.
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        - key: kubernetes.io/os
          operator: In
          values: ["linux"]

      # Nodes expire after 30 days regardless of load.
      # This forces periodic replacement, ensuring nodes run recent OS patches
      # and AMI updates. Karpenter drains and replaces the node gracefully.
      expireAfter: 720h

  disruption:
    # WhenEmptyOrUnderutilized: Karpenter considers all nodes.
    # If a node is empty OR if pods could be consolidated onto fewer nodes
    # without violating any scheduling constraints, Karpenter will drain
    # and terminate the underutilized node.
    consolidationPolicy: WhenEmptyOrUnderutilized

    # Wait 60 seconds after a scaling event before reconsidering consolidation.
    # This prevents thrashing during rapid scale-up/scale-down cycles.
    consolidateAfter: 60s

    # Disruption budgets control how many nodes can be disrupted simultaneously.
    # 10% means at most 10% of the pool's nodes can be draining at any moment.
    # The second budget blocks all disruption during business hours (Mon–Fri 9am–5pm).
    budgets:
      - nodes: "10%"
      - schedule: "0 9 * * mon-fri"
        duration: 8h
        nodes: "0"

  # Pool-level resource cap. Karpenter stops provisioning new nodes once
  # the pool's aggregate CPU/memory exceeds these values.
  # This is a cost safeguard, not a hard quota — running pods are not evicted
  # when the limit is crossed; only new provisioning is blocked.
  limits:
    cpu: "100"
    memory: 400Gi
EOF
```

### Understanding the requirements intersection

When a pod arrives with `resources.requests.cpu: 2` and `resources.requests.memory: 4Gi`, Karpenter:

1. Filters the allowed instance families to `t4g`, `m7g`, `m8g`
2. Filters out `nano` and `micro`
3. Queries EC2 for current Spot and on-demand pricing for all remaining instances in the AZs covered by the tagged subnets
4. Selects the cheapest instance that fits the pod's requests plus the overhead for DaemonSets and kubelet reservations
5. If the pod also has a `topologySpreadConstraint` requiring placement in a specific AZ, Karpenter factors that in too

For the three families in this tutorial, Karpenter would typically pick `t4g.medium` (2 vCPU, 4GiB) for the example request. If the workload needed 8 vCPU, it would consider `t4g.2xlarge`, `m7g.2xlarge`, and `m8g.2xlarge` and select whichever is cheapest and available.

> **Alternative:** Set `values: ["on-demand"]` only if your workload cannot tolerate interruption. On-demand instances are never reclaimed by AWS mid-run. Spot instances save 60–90% of the on-demand price but can be interrupted with a two-minute warning when AWS needs the capacity back. Stateless, re-runnable workloads (batch jobs, CI runners, web servers with graceful shutdown) are good candidates for Spot. Stateful databases and long-running sequential jobs are not.

### Verify the NodePool

```bash
kubectl get nodepool graviton
```

`READY` should be `True`. If it is not, check the `EC2NodeClass` readiness first — the `NodePool` inherits its readiness from the referenced class.

---

## 🔧 Part 7: Observing Dynamic Provisioning

### Deploy a test workload

This deployment requests more resources than a single existing system node can provide, which forces Karpenter to provision a new node:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      # Target Graviton nodes explicitly.
      # Remove this nodeSelector to let Karpenter choose between pools.
      nodeSelector:
        eks-node-pool: graviton
      containers:
        - name: inflate
          image: public.ecr.aws/amazonlinux/amazonlinux:2023
          command: ["/bin/sh", "-c", "sleep infinity"]
          resources:
            requests:
              # Request enough to force a new node on a small system pool
              cpu: "2"
              memory: "3Gi"
EOF
```

### Watch Karpenter provision a node

Open two terminal windows.

**Terminal 1 — watch Karpenter logs in real time:**

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f
```

**Terminal 2 — watch node and pod events:**

```bash
# Watch for new nodes appearing
kubectl get nodes -w

# In a separate pane: watch the pod move from Pending to Running
kubectl get pods -w
```

Expected sequence:

1. Pod status: `Pending` (no node fits its requests)
2. Karpenter log: `computed new nodeclaim` showing the selected instance type
3. Karpenter log: `launched nodeclaim` with the EC2 instance ID
4. Node status: `NotReady` (node registered, joining)
5. Node status: `Ready` (~60 seconds from launch)
6. Pod status: `Running`

### Scale up to trigger more provisioning

```bash
# Scale to 5 replicas — each requesting 2 CPU and 3Gi memory
kubectl scale deployment inflate --replicas=5

# Watch nodes appear; Karpenter will bin-pack where possible
# and provision additional nodes where needed
kubectl get nodes -w
```

Karpenter performs bin-packing before launching new nodes: it will schedule as many of the new replicas as possible onto the already-provisioned node from the previous step. Only the pods that do not fit trigger additional EC2 launches.

### Scale down and observe consolidation

```bash
kubectl scale deployment inflate --replicas=0
```

Wait 60–90 seconds (the `consolidateAfter: 60s` from the NodePool, plus the node drain time). Karpenter will:

1. Detect that the node is now empty
2. Taint the node with `karpenter.sh/disruption: NoSchedule`
3. Issue `kubectl drain` (cordon + pod eviction)
4. Call `ec2:TerminateInstances`
5. Remove the Node object from the cluster

```bash
# Confirm the node is gone
kubectl get nodes
```

> **Concept:** `WhenEmptyOrUnderutilized` consolidation works in two modes: *empty node deletion* (fastest, no pod movement needed) and *node replacement* (Karpenter determines that two nodes' workloads could fit on one smaller node, provisions the replacement, migrates pods, then terminates the originals). Both modes respect `PodDisruptionBudgets` and `do-not-disrupt` annotations.

---

## 🔧 Troubleshooting

### Pods stay Pending; no node is provisioned

**Symptom:** Pod shows `Pending` for more than 2 minutes; no new node appears.

**Cause:** Karpenter cannot find a valid provisioning path. Common reasons:

1. No matching `NodePool` — check `kubectl get nodepool` and confirm the pool is `READY: True`
2. EC2NodeClass not ready — check `kubectl describe ec2nodeclass graviton` for condition messages
3. Pod's `nodeSelector` or affinity rules exclude all NodePool instances
4. NodePool limits reached — check `kubectl get nodepool graviton -o jsonpath='{.status.resources}'`
5. IAM permission error — check Karpenter logs for `AccessDenied`

```bash
# Check Karpenter for errors
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter | grep -E "ERROR|error|denied"
```

### Node joins but pod stays Pending with ImagePullBackOff

**Symptom:** Node is `Ready`; pod shows `ImagePullBackOff`.

**Cause:** The container image does not have a `linux/arm64` manifest. The Graviton node cannot run an `amd64`-only image.

**Resolution:** Rebuild and push a multi-arch image (see Part 1), or add a `nodeSelector: kubernetes.io/arch: amd64` to the pod to route it to a non-Graviton node.

### Karpenter logs show "unable to find instance type"

**Symptom:** `failed to create node` with a message about no available instance types.

**Cause:** The intersection of NodePool requirements and EC2 availability in the selected region/AZ is empty. This can happen if the requested instance families are not available in all AZs in the region.

**Resolution:** Broaden the `instance-family` or `instance-size` requirements, or add more AZs:

```bash
# Check which instance types are available in your region
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=m8g.*" \
  --region "${AWS_REGION}" \
  --query "InstanceTypeOfferings[].{Type:InstanceType,AZ:Location}" \
  --output table
```

### Node not terminating after scale-down

**Symptom:** Empty node stays in the cluster for more than 5 minutes after all pods are removed.

**Cause:** A DaemonSet pod is still on the node (DaemonSet pods do not block consolidation, but their presence means the node is not truly empty), or the `consolidateAfter` timer has not elapsed, or a disruption budget is blocking the action.

**Resolution:** Check active budgets:

```bash
kubectl describe nodepool graviton | grep -A5 Disruption
```

Check for `do-not-disrupt` annotations on pods:

```bash
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.metadata.annotations["karpenter.sh/do-not-disrupt"] == "true") | .metadata.name'
```

---

## 📚 Further Reading

- [Karpenter NodePools reference](https://karpenter.sh/docs/concepts/nodepools/) — full `NodePool` spec with all fields documented
- [Karpenter EC2NodeClasses reference](https://karpenter.sh/docs/concepts/nodeclasses/) — full `EC2NodeClass` spec
- [Karpenter Disruption concepts](https://karpenter.sh/docs/concepts/disruption/) — deep dive on consolidation, expiry, and drift
- [AWS Graviton processor comparison](https://aws.amazon.com/ec2/graviton/) — official AWS comparison of Graviton generations
- [EKS Best Practices Guide – Cost Optimization](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_compute/) — covers Spot strategy, Karpenter tuning, and multi-arch images in production
