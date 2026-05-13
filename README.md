#  Deploy 2048 Game on Amazon EKS Fargate with AWS Load Balancer Controller

This project demonstrates how to deploy the **2048 game application** on **Amazon EKS using AWS Fargate** and expose it publicly using the **AWS Load Balancer Controller (ALB Ingress)**.

This project covers:

* Creating an EKS cluster using Fargate
* Configuring kubectl access
* Creating Fargate profiles
* Deploying a sample application
* Configuring IAM OIDC provider
* Creating IAM service accounts
* Installing cert-manager
* Tagging public subnets
* Installing AWS Load Balancer Controller
* Accessing application through ALB

---

# 📌 Architecture

```plaintext
User
   ↓
Application Load Balancer (ALB)
   ↓
Kubernetes Ingress
   ↓
Kubernetes Service
   ↓
2048 Application Pods
   ↓
Amazon EKS Cluster
   ↓
AWS Fargate
```

---

# 🛠️ AWS Services Used

| Service      | Purpose                           |
| ------------ | --------------------------------- |
| Amazon EKS   | Managed Kubernetes service        |
| AWS Fargate  | Serverless compute for containers |
| AWS IAM      | Permission management             |
| AWS ALB      | Public application access         |
| Amazon VPC   | Networking                        |
| cert-manager | Webhook TLS certificates          |
| Helm         | Install ALB controller            |

---

# 📋 Prerequisites

Install the following tools before starting:

---

## 1. AWS CLI

Verify installation:

```bash
aws --version
```

Install from:

[AWS CLI Documentation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html?utm_source=chatgpt.com)

Configure AWS credentials:

```bash
aws configure
```

Enter:

* AWS Access Key
* AWS Secret Key
* Default region
* Output format

---

## 2. eksctl

Verify installation:

```bash
eksctl version
```

Install from:

[eksctl Official Documentation](https://eksctl.io/installation/?utm_source=chatgpt.com)

---

## 3. kubectl

Verify installation:

```bash
kubectl version --client
```

Install from:

[kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/?utm_source=chatgpt.com)

---

## 4. Helm

Verify installation:

```bash
helm version
```

Install from:

[Helm Official Website](https://helm.sh/docs/intro/install/?utm_source=chatgpt.com)
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 03 33 PM" src="https://github.com/user-attachments/assets/3e38052f-0b47-4cb7-8b5f-312df4542bcc" />

---

# Step 1: Create EKS Cluster

Create an EKS cluster with Fargate support:

```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```

### What this creates:

* EKS Cluster
* VPC
* Subnets
* Security Groups
* Fargate support

Expected time: **10–15 minutes**
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 02 04 PM" src="https://github.com/user-attachments/assets/49958050-dd8e-48f4-b099-88a7d5e55bca" />
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 55 35 PM" src="https://github.com/user-attachments/assets/1ccb1412-ca15-4234-9b8f-6d3efdd1bf0d" />


---

# Step 2: Configure kubectl

Connect your local machine to the EKS cluster:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

Verify cluster connection:

```bash
kubectl get nodes
```

Since this project uses Fargate, traditional worker nodes may not appear.
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 05 20 PM" src="https://github.com/user-attachments/assets/42b2193a-c5a4-4687-a0dc-8b3b36bf48ab" />


---

# Step 3: Create Fargate Profile for Application Namespace

Create a Fargate profile for the `game-2048` namespace:

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

This ensures your application pods run on Fargate.
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 55 49 PM" src="https://github.com/user-attachments/assets/cf9415f5-7dba-4cf5-8379-8a2555db1bfd" />

---

# Step 4: Deploy 2048 Application

Deploy the sample application:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

This creates:

* Namespace
* Deployment
* Service
* Ingress

Verify:

```bash
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 15 48 PM" src="https://github.com/user-attachments/assets/7254a58a-c753-428a-806f-0cb274bbf263" />


At this point:

```plaintext
ADDRESS = empty
```

This is expected because the ALB controller is not installed yet.

2048

---

# Step 5: Create OIDC Provider

This allows Kubernetes service accounts to assume IAM roles.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --approve \
  --region us-east-1
```

# Step 6: Create IAM Service Account

⚠️ Important:

Use the **AWS managed policy**

**Do NOT download custom JSON policies**

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

Verify:

```bash
kubectl describe sa aws-load-balancer-controller -n kube-system
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 27 58 PM" src="https://github.com/user-attachments/assets/b5d346c6-dc3a-4b76-8d29-b3a92d6ffa68" />

---

# Step 7: Install cert-manager

This step is mandatory for webhook TLS.

Without this, you'll get certificate errors.

Install:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
```

Wait until ready:

```bash
kubectl wait \
--for=condition=available \
deployment/cert-manager-webhook \
-n cert-manager \
--timeout=300s
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 48 00 PM" src="https://github.com/user-attachments/assets/96052b61-2047-40c6-971c-e4f7fd2b4095" />

---

# Step 8: Tag Public Subnets via CLI

This step is required.

Without subnet tags, ALB controller cannot identify where to create public load balancers.

---

## Get VPC ID

```bash
VPC_ID=$(aws eks describe-cluster \
--name demo-cluster \
--query "cluster.resourcesVpcConfig.vpcId" \
--output text \
--region us-east-1)
```

---

## Get Public Subnets

```bash
PUBLIC_SUBNETS=$(aws ec2 describe-subnets \
--filters "Name=vpc-id,Values=$VPC_ID" \
"Name=map-public-ip-on-launch,Values=true" \
--query "Subnets[*].SubnetId" \
--output text \
--region us-east-1)
```

---

## Tag Public Subnets

```bash
aws ec2 create-tags \
  --resources $PUBLIC_SUBNETS \
  --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/demo-cluster,Value=shared \
  --region us-east-1
```

---

# Step 9: Install AWS Load Balancer Controller

Add Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Update repository:

```bash
helm repo update
```

Install controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 11 04 09 PM" src="https://github.com/user-attachments/assets/47a359bc-727a-4d1e-9526-e5adf19c8866" />

---

## Verify Controller

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Wait until:

```plaintext
AVAILABLE = 1
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 56 09 PM" src="https://github.com/user-attachments/assets/0745caa2-cbc0-47ec-8f1b-04458b947811" />

---

# Step 10: Get ALB URL

Watch ingress:

```bash
kubectl get ingress -n game-2048 -w
```

After 2–3 minutes:

```plaintext
xxxxxxxxxx.us-east-1.elb.amazonaws.com
```
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 10 58 49 PM" src="https://github.com/user-attachments/assets/c8594a4d-c40e-4eba-93a7-8e23ed776259" />
<img width="1710" height="1107" alt="Screenshot 2026-05-13 at 11 03 37 PM" src="https://github.com/user-attachments/assets/58815928-f9c5-49ef-9faf-7eea9727a7df" />

Copy the URL and open it in your browser.

Your application is now live 🎉

---

# Why These Extra Steps Are Mandatory

| Step               | Why It Matters                                                     |
| ------------------ | ------------------------------------------------------------------ |
| AWS Managed Policy | Custom JSON policies are outdated and missing required permissions |
| cert-manager       | Required for webhook TLS certificates                              |
| Subnet Tags        | ALB controller uses these tags to identify public subnets          |

---

# Verification Commands

Check pods:

```bash
kubectl get pods -n game-2048
```

Check ingress:

```bash
kubectl get ingress -n game-2048
```

Check ALB controller:

```bash
kubectl get pods -n kube-system
```

---

# Troubleshooting

---

## Ingress ADDRESS remains empty

Run:

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

---

## Check ALB Controller Logs

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=50
```

---

## Common Errors

### No subnets found

You skipped subnet tagging.

Repeat Step 8.

---

### Certificate errors

Reinstall cert-manager.

---

### IAM permission errors

Ensure you're using:

```plaintext
AWSLoadBalancerControllerIAMPolicy
```

---

# Cleanup Resources

Delete all resources after testing:

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

This removes:

* EKS Cluster
* Fargate Profiles
* ALB
* VPC
* Networking resources

---

# Project Structure

```plaintext
eks-fargate-alb-project/
│
├── README.md
├── screenshots/
│   ├── cluster.png
│   ├── deployment.png
│   ├── alb-controller.png
│   └── output.png
```

---

# Learning Outcomes

✅ EKS Cluster Deployment
✅ Kubernetes on Fargate
✅ IAM Role Configuration
✅ ALB Ingress Setup
✅ cert-manager Integration
✅ Networking Configuration
✅ Kubernetes Troubleshooting

---

# Author

**Amtul Saboor**
DevOps & Cloud Engineer 
---

# License

This project is licensed under the MIT License.

---

## ⭐ If this project helped you, don’t forget to star the repository!
