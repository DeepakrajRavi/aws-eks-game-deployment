# Deploying a Simple Game on AWS EKS with ALB Ingress Controller

This project demonstrates how to deploy a simple containerized game application on AWS Elastic Kubernetes Service (EKS) using Fargate worker nodes and AWS Application Load Balancer (ALB) Ingress Controller for traffic management.

---

## Table of Contents

- [Why AWS EKS?](#why-aws-eks)
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Setup AWS Environment](#setup-aws-environment)
  - [Create AWS Account and IAM Users](#create-aws-account-and-iam-users)
  - [Configure AWS CLI and kubectl](#configure-aws-cli-and-kubectl)
  - [Prepare Networking and Security](#prepare-networking-and-security)
- [EKS Cluster and Node Setup](#eks-cluster-and-node-setup)
- [Deploy Sample Game Application](#deploy-sample-game-application)
- [Setup AWS Load Balancer Controller](#setup-aws-load-balancer-controller)
- [Commands Summary](#commands-summary)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Why AWS EKS?

- **Managed Control Plane:** AWS manages Kubernetes control plane components, upgrades, patches, and ensures high availability.
- **Automated Updates & Scalability:** EKS automatically updates Kubernetes versions and scales control plane as needed.
- **AWS Integration:** Seamlessly integrates with AWS services such as IAM, VPC, and Load Balancers.
- **Security & Compliance:** Built-in security features and compliance certifications.
- **Monitoring & Logging:** Integrates with AWS CloudWatch for cluster health and metrics.
- **Community Support:** Backed by continuous improvements from the Kubernetes community.

**Note:** While EKS provides automation and ease, it comes with higher cost and less control over infrastructure compared to self-managed Kubernetes.

---

## Prerequisites

Ensure the following tools are installed and configured on your local machine:

- **kubectl:** CLI for interacting with Kubernetes clusters  
  [Install or Update kubectl](https://kubernetes.io/docs/tasks/tools/)

- **eksctl:** CLI to create and manage EKS clusters  
  [Install or Update eksctl](https://eksctl.io/introduction/installation/)

- **AWS CLI:** CLI for AWS services management  
  [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)  
  Configure with:  
  ```bash
  aws configure
  ```

---

## Architecture Overview

- EKS control plane is fully managed by AWS.
- Worker nodes run either on EC2 instances or AWS Fargate (serverless).
- Pods deployed in worker nodes located in private subnets within an AWS VPC.
- Service exposure via Kubernetes NodePort and AWS ALB Ingress Controller.
- AWS ALB Ingress Controller automatically creates and manages an Application Load Balancer in public subnet to route external traffic to the pods through Ingress resources.

---

## Setup AWS Environment

### 1. Create AWS Account and IAM Users

- Register at [aws.amazon.com](https://aws.amazon.com/).
- Create IAM users with necessary permissions.
- Enable MFA for security.
- Generate access keys for programmatic access.

### 2. Configure AWS CLI and kubectl

```bash
aws configure
```

- Install `kubectl` and configure it to connect with your EKS cluster.

Update kubeconfig for your cluster:

```bash
aws eks update-kubeconfig --name <your-cluster-name>
```

Test access:

```bash
kubectl get nodes
```

### 3. Prepare Networking and Security Groups

- Create a VPC with public and private subnets.
- Configure Security Groups for control of inbound/outbound traffic.
- Setup Internet Gateway (IGW) attached to VPC for internet access.
- Define IAM policies granting permissions for EKS worker nodes and ALB controller.

---

## EKS Cluster and Node Setup

- Create your EKS cluster using `eksctl` or AWS Console.
- Choose EC2 or Fargate for worker nodes.
- Ensure nodes are in private subnets.
- Setup IAM OIDC provider for your cluster:

```bash
export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

---

## Deploy Sample Game Application

### Deployment Manifest (`deploy.yml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-sample-linux-deployment
  labels:
    app: eks-sample-linux-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eks-sample-linux-app
  template:
    metadata:
      labels:
        app: eks-sample-linux-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
                - arm64
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:1.23
        ports:
        - name: http
          containerPort: 80
        imagePullPolicy: IfNotPresent
      nodeSelector:
        kubernetes.io/os: linux
```

Apply deployment:

```bash
kubectl apply -f deploy.yml
```

### Service Manifest (`service.yml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eks-sample-linux-service
  labels:
    app: eks-sample-linux-app
spec:
  selector:
    app: eks-sample-linux-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply service:

```bash
kubectl apply -f service.yml
```

---

## Setup AWS Load Balancer Controller

1. **Download IAM Policy**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

2. **Create IAM Policy**

```bash
aws iam create-policy     --policy-name AWSLoadBalancerControllerIAMPolicy     --policy-document file://iam_policy.json
```

3. **Create IAM Role and Service Account**

```bash
eksctl create iamserviceaccount   --cluster=<your-cluster-name>   --namespace=kube-system   --name=aws-load-balancer-controller   --role-name AmazonEKSLoadBalancerControllerRole   --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy   --approve
```

4. **Add and Update Helm Repository**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

5. **Install AWS Load Balancer Controller**

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=<your-cluster-name>   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller   --set region=<your-region>   --set vpcId=<your-vpc-id>
```

6. **Verify Installation**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Commands Summary

| Task                            | Command Example                                                                                          |
|--------------------------------|--------------------------------------------------------------------------------------------------------|
| Create EKS cluster              | `eksctl create cluster --name demo-cluster --region us-east-1 --fargate`                                |
| Update kubeconfig               | `aws eks update-kubeconfig --name demo-cluster`                                                        |
| Deploy app                     | `kubectl apply -f deploy.yml`                                                                           |
| Deploy service                 | `kubectl apply -f service.yml`                                                                          |
| Download ALB IAM policy         | `curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json` |
| Create IAM policy               | `aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json` |
| Create IAM service account      | `eksctl create iamserviceaccount --cluster demo-cluster --namespace kube-system --name aws-load-balancer-controller --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy --approve` |
| Install ALB Controller Helm     | `helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=demo-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=vpc-xxxx` |
| Verify deployments              | `kubectl get deployment -n kube-system`                                                                 |

---

## Troubleshooting

- **IAM Role Creation Errors:** Verify your AWS account ID and policy ARNs are correct.  
- **ALB Controller Deployment Pending:** Check pod logs with `kubectl logs -n kube-system deployment/aws-load-balancer-controller`.  
- **Pods Not Starting:** Verify node capacity and cluster health.  
- **Service Not Exposed:** Check if Ingress and ALB are properly created and associated with public subnet.

---

## References

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)  
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)  
- [eksctl Documentation](https://eksctl.io/)  
- [kubectl Official Docs](https://kubernetes.io/docs/reference/kubectl/overview/)  

---

Feel free to reach out if you encounter any issues or have suggestions for improvements!

---

*© 2025 Deepakraj Ravi*
