# Cloud-Native Game Deployment on AWS EKS 🕹
<div align="center">

  <img width="1920" height="1080" alt="cloudnativeGame" src="https://github.com/user-attachments/assets/6cbd93d9-e114-4b8c-b5b8-d4b801565c39" />

  <h2 align="center">Cloud-Native Game Deployment on AWS EKS 🕹</h2>

  A cloud-native web application deployment using Docker, AWS EKS & Fargate. <br> Responsive for both desktops and mobile devices. <br> Made with the intention of learning the in's and out's of Kubernetes 😁. 

</div>

## Project Video 🎞


https://github.com/user-attachments/assets/d3578b5f-584f-4feb-9af0-544e65961485


## Project Summary
This project demonstrates the end-to-end process of deploying a containerized web application (the classic 2048 game) onto a scalable, secure, and serverless Kubernetes cluster on AWS. The entire infrastructure is provisioned and managed using modern Infrastructure as Code (IaC) and cloud-native best practices.

## Core Technologies
- Containerization: Docker
- Orchestration: Kubernetes
- Managed Kubernetes Service: Amazon EKS (Elastic Kubernetes Service)
- Serverless Compute: AWS Fargate
- Networking: Amazon VPC (Virtual Private Cloud)
- Load Balancing: AWS Application Load Balancer (ALB)
- Infrastructure as Code (IaC): AWS CloudFormation (via eksctl), kubectl YAML
- Security & Permissions: AWS IAM (Identity and Access Management)
- Package Management: Helm

## :hammer_and_wrench: Architecture:
The architecture is designed for high availability and security, with a public-facing load balancer directing traffic to a secure, private application backend running on a serverless compute engine.
<div align="center">
  
  ![CNGChart](https://github.com/user-attachments/assets/780e6ff3-17e8-4028-bfc9-fe47febb5cda)

</div>

## Usage ⚙:
Just click on the link and enjoy! <br>
***⚠ NOTE: Make sure the appliaction begins with `http://` as `https://` is currently not configured. Most mobile phones force web application to `https`, so keep it in mind! The application works for both desktops and mobile devices.***

## Prerequisites
Before you begin, ensure you have the following tools installed and configured:
- AWS CLI
- `eksctl`
- `kubectl`
- `helm`

## Step-by-Step Deployment Guide

### 1. Create the EKS Cluster with Fargate

This command provisions a new EKS cluster with a default Fargate profile. eksctl automates the creation of the VPC, subnets, control plane, and other necessary resources.

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

After the cluster is created, update your local kubeconfig to interact with it:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

### 2. Create a Dedicated Fargate Profile for the Application

Create a specific Fargate profile for the game-2048 namespace to ensure our application pods are scheduled onto the serverless Fargate infrastructure.

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### 3. Deploy the 2048 Game Application

Apply the Kubernetes manifest for the 2048 game. This single file creates the namespace, deployment, service, and ingress resources needed to run the application.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Verify that the pods are running:

```bash
# Check the initial status (they will be Pending at first)
kubectl get pods -n game-2048

# Watch the pods until they are in the "Running" state
kubectl get pods -n game-2048 -w
```

### 4. Configure IAM for the AWS Load Balancer Controller

The AWS Load Balancer Controller needs permissions to manage ALBs on your behalf. We will create an IAM OIDC provider and an IAM role attached to a Kubernetes service account.

#### a. Create an IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

#### b. Create the IAM Policy

Download the policy JSON file from AWS:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Create the policy in IAM:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

**Note:** Remember your AWS Account ID from the output.

#### c. Create the IAM Service Account

This command creates an IAM role and a Kubernetes service account (aws-load-balancer-controller in the kube-system namespace) and binds them together.

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Note:** Replace `YOUR_AWS_ACCOUNT_ID` with your actual AWS Account ID.

### 5. Install the AWS Load Balancer Controller with Helm

With the permissions configured, install the controller using Helm.

#### a. Add the EKS charts repository

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

#### b. Install the Helm chart

This command installs the controller, tells it which cluster it's managing, and instructs it to use the service account we created in the previous step.

```bash
# Get your VPC ID
VPC_ID=$(aws eks describe-cluster --name demo-cluster --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID
```

### 6. Verify the Deployment

Check that the AWS Load Balancer Controller deployment is ready:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Wait until READY shows 2/2.

Finally, check the ingress object. The ADDRESS field will eventually be populated with the DNS name of the Application Load Balancer:

```bash
kubectl get ingress -n game-2048
```

This can take a few minutes.

## Accessing the Application

Once the ADDRESS field in the ingress is populated, copy that DNS name and paste it into your browser to play the game!

**Note:** The deployment is on HTTP. If you have trouble connecting from a mobile device, make sure the URL starts with `http://` as most phones now default to HTTPS.

## Cleanup

To avoid ongoing charges, remember to delete the resources when you're done:

```bash
# Delete the cluster (this will also delete the Fargate profiles and load balancer)
eksctl delete cluster --name demo-cluster --region us-east-1
```
