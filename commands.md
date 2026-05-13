# EKS Fargate + ALB Controller Setup Commands

This file contains all commands used in the project in exact execution order.

---

## Step 1: Verify AWS CLI Installation

```bash
aws --version
```

---

## Step 2: Configure AWS Credentials

```bash
aws configure
```

---

## Step 3: Verify eksctl Installation

```bash
eksctl version
```

---

## Step 4: Verify kubectl Installation

```bash
kubectl version --client
```

---

## Step 5: Verify Helm Installation

```bash
helm version
```

---

# Step 6: Create EKS Cluster with Fargate

```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```

---

# Step 7: Configure kubectl for EKS Cluster

```bash
aws eks update-kubeconfig \
  --name demo-cluster \
  --region us-east-1
```

---

# Step 8: Verify Cluster Connection

```bash
kubectl get nodes
```

---

# Step 9: Create Fargate Profile

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

---

# Step 10: Deploy 2048 Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

2048

---

# Step 11: Verify Application Deployment

```bash
kubectl get pods -n game-2048
```

```bash
kubectl get svc -n game-2048
```

```bash
kubectl get ingress -n game-2048
```

---

# Step 12: Create OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --approve \
  --region us-east-1
```

---

# Step 13: Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::aws:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve \
  --region us-east-1
```

---

# Step 14: Verify IAM Service Account

```bash
kubectl describe sa aws-load-balancer-controller -n kube-system
```

---

# Step 15: Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
```

---

# Step 16: Wait for cert-manager

```bash
kubectl wait \
  --for=condition=available \
  deployment/cert-manager-webhook \
  -n cert-manager \
  --timeout=300s
```

---

# Step 17: Get VPC ID

```bash
VPC_ID=$(aws eks describe-cluster \
  --name demo-cluster \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text \
  --region us-east-1)
```

---

# Step 18: Get Public Subnets

```bash
PUBLIC_SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=map-public-ip-on-launch,Values=true" \
  --query "Subnets[*].SubnetId" \
  --output text \
  --region us-east-1)
```

---

# Step 19: Tag Public Subnets

```bash
aws ec2 create-tags \
  --resources $PUBLIC_SUBNETS \
  --tags Key=kubernetes.io/role/elb,Value=1 \
         Key=kubernetes.io/cluster/demo-cluster,Value=shared \
  --region us-east-1
```

---

# Step 20: Add Helm Repository

```bash
helm repo add eks https://aws.github.io/eks-charts
```

---

# Step 21: Update Helm Repository

```bash
helm repo update
```

---

# Step 22: Install AWS Load Balancer Controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID
```

---

# Step 23: Verify ALB Controller

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

# Step 24: Get ALB URL

```bash
kubectl get ingress -n game-2048 -w
```

---

# Step 25: Verify Running Resources

```bash
kubectl get pods -n game-2048
```

```bash
kubectl get svc -n game-2048
```

```bash
kubectl get ingress -n game-2048
```

```bash
kubectl get pods -n kube-system
```

---

# Step 26: Troubleshooting Commands

### Describe Ingress

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

### Check ALB Controller Logs

```bash
kubectl logs \
-n kube-system deployment/aws-load-balancer-controller \
--tail=50
```

---

# Step 27: Cleanup Everything

```bash
eksctl delete cluster \
  --name demo-cluster \
  --region us-east-1
```

---

## Final Output

After completing all commands successfully, you’ll receive an ALB URL like:

```bash
xxxxxxxx.us-east-1.elb.amazonaws.com
```

Open it in your browser to access the deployed **2048 game application.**

---

## File Structure Example

```bash
project/
├── README.md
└── commands.md
```
