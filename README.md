# EKS Cluster with Autoscaling

This project demonstrates how to set up an Amazon EKS cluster with automatic scaling (Cluster Autoscaler) in AWS.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation and Setup](#installation-and-setup)
- [Project Components](#project-components)
- [Testing and Verification](#testing-and-verification)
- [Cleanup](#cleanup)

## 🎯 Overview

This project contains configurations for deploying Kubernetes Cluster Autoscaler in Amazon EKS. The autoscaler automatically adjusts the number of nodes in the cluster based on workload demands.

### Key Features:

- ✅ Automatic scaling of EKS nodes
- ✅ Integration with AWS Auto Scaling Groups
- ✅ IRSA (IAM Roles for Service Accounts) implementation
- ✅ Auto-discovery of node groups via tags
- ✅ Balancing of similar node groups
- ✅ NGINX test application with LoadBalancer

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    AWS Cloud                        │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │              EKS Cluster                      │ │
│  │                                               │ │
│  │  ┌──────────────────────────────────────┐    │ │
│  │  │   kube-system namespace              │    │ │
│  │  │                                      │    │ │
│  │  │   ┌──────────────────────────────┐  │    │ │
│  │  │   │  Cluster Autoscaler Pod      │  │    │ │
│  │  │   │  ┌────────────────────────┐  │  │    │ │
│  │  │   │  │ IRSA Role Integration  │  │  │    │ │
│  │  │   │  └────────────────────────┘  │  │    │ │
│  │  │   └──────────────────────────────┘  │    │ │
│  │  │                │                    │    │ │
│  │  └────────────────┼─────────────────────┘    │ │
│  │                   │                          │ │
│  │  ┌────────────────▼───────────────────┐     │ │
│  │  │   Worker Nodes (Auto Scaling)      │     │ │
│  │  │                                    │     │ │
│  │  │   ┌─────────┐  ┌─────────┐        │     │ │
│  │  │   │ NGINX   │  │  NGINX  │  ...   │     │ │
│  │  │   │  Pod    │  │   Pod   │        │     │ │
│  │  │   └─────────┘  └─────────┘        │     │ │
│  │  └────────────────────────────────────┘     │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐   │
│  │  Auto Scaling    │◄────►│  IAM Role with   │   │
│  │  Groups          │      │  Custom Policy   │   │
│  └──────────────────┘      └──────────────────┘   │
└─────────────────────────────────────────────────────┘
```

## 📦 Prerequisites

### Tools

- [AWS CLI](https://aws.amazon.com/cli/) v2.x
- [eksctl](https://eksctl.io/) v0.150+
- [kubectl](https://kubernetes.io/docs/tasks/tools/) v1.28+
- [helm](https://helm.sh/) (optional)

### AWS Resources

- AWS account with appropriate permissions
- Configured AWS profile (`aws configure`)
- VPC with public and private subnets (created automatically via eksctl)

## 🚀 Installation and Setup

### Step 1: Create EKS Cluster

```bash
eksctl create cluster \
  --name eks-cluster-test \
  --region ap-southeast-2 \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

### Step 2: Create IAM Policy for Cluster Autoscaler

```bash
aws iam create-policy \
  --policy-name EKSClusterAutoscalerPolicy \
  --policy-document file://custom-policy.json
```

Save the ARN of the created policy.

### Step 3: Create IAM Role with IRSA

```bash
eksctl create iamserviceaccount \
  --cluster=eks-cluster-test \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/EKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### Step 4: Add Tags to Auto Scaling Group

```bash
aws autoscaling create-or-update-tags \
  --tags \
    ResourceId=<YOUR_ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=false \
    ResourceId=<YOUR_ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/eks-cluster-test,Value=owned,PropagateAtLaunch=false
```

### Step 5: Update Cluster Autoscaler Configuration

Edit `cluster-autoscaler-autodiscover.yaml`:

1. Replace `YOUR_ACCOUNT_ID` with your AWS Account ID (line 12)
2. Specify your region in the `AWS_REGION` variable (line 157)
3. Specify your cluster name in the `--node-group-auto-discovery` parameter (line 172)

### Step 6: Deploy Cluster Autoscaler

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

### Step 7: Check Status

```bash
kubectl get deployment cluster-autoscaler -n kube-system
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

### Step 8: Deploy Test Application

```bash
kubectl apply -f nginx.yaml
```

## 📁 Project Components

### `cluster-autoscaler-autodiscover.yaml`

Main manifest for deploying Cluster Autoscaler, includes:

- **ServiceAccount** with IRSA role annotation
- **RBAC rules** (ClusterRole, Role, Bindings)
- **Deployment** with autoscaler container

**Key Parameters:**

- `--node-group-auto-discovery` - automatic ASG discovery via tags
- `--balance-similar-node-groups` - balancing of similar node groups
- `--skip-nodes-with-system-pods=false` - allows removal of nodes with system pods
- Image: `registry.k8s.io/autoscaling/cluster-autoscaler:v1.34.2`

### `custom-policy.json`

IAM policy with minimum required permissions for Cluster Autoscaler:

- Read information about Auto Scaling Groups
- Modify desired instance count
- Terminate instances in ASG
- Read information about EC2 instance types

### `nginx.yaml`

Test application for verifying autoscaling:

- Deployment with 1 NGINX replica
- LoadBalancer Service for external access

### `commands.md`

Reference information for setup and commands.

## 🔍 Testing and Verification

### Testing Scale-Up

Increase the number of NGINX replicas to create load:

```bash
kubectl scale deployment nginx --replicas=50
```

Watch for new nodes being created:

```bash
kubectl get nodes -w
```

Check autoscaler logs:

```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system | grep -i "scale"
```

### Testing Scale-Down

Reduce the load:

```bash
kubectl scale deployment nginx --replicas=1
```

After 10-15 minutes, excess nodes will be automatically removed.

### Monitoring Events

```bash
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep cluster-autoscaler
```

### Check ASG Status

```bash
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?Tags[?Key=='k8s.io/cluster-autoscaler/eks-cluster-test']].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]" \
  --output table
```

## 🧹 Cleanup

```bash
# Delete test application
kubectl delete -f nginx.yaml

# Delete Cluster Autoscaler
kubectl delete -f cluster-autoscaler-autodiscover.yaml

# Delete EKS cluster
eksctl delete cluster --name eks-cluster-test --region ap-southeast-2

# Delete IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/EKSClusterAutoscalerPolicy
```

## 📚 Additional Resources

- [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [EKS Workshop](https://www.eksworkshop.com/)
- [Cluster Autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)

## 🔧 Troubleshooting

### Autoscaler is not scaling the cluster

1. Check logs:
   ```bash
   kubectl logs deployment/cluster-autoscaler -n kube-system
   ```

2. Check IRSA role:
   ```bash
   kubectl describe sa cluster-autoscaler -n kube-system
   ```

3. Check ASG tags:
   ```bash
   aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[*].Tags"
   ```

### Pending pods do not trigger scaling

- Ensure that the ASG has correct `MinSize` and `MaxSize` values
- Verify that pods do not have nodeSelector/affinity that cannot be satisfied
- Check for Pod Disruption Budgets (PDB)

### Slow scale-down

- Default wait time before removing a node: 10 minutes
- Nodes with certain pods will not be removed (can be configured via annotations)

## 📝 License

This project is created for educational purposes.

---

**Region:** ap-southeast-2 (Sydney)  
**Cluster Autoscaler Version:** v1.34.2  
**Kubernetes Version:** 1.28+
