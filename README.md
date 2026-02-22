# 🚀 3-Tier Application on AWS EKS

> **Deploy a production-ready React + Flask + PostgreSQL (RDS) application on Amazon EKS with ALB Ingress, Route 53, OIDC & IAM.**

![banner](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*PB6jgC5b4bO0T3F-InbOLw.png)

---

## 📑 Table of Contents

| # | Section |
|---|---------|
| 1 | [Architecture Overview](#-architecture-overview) |
| 2 | [Prerequisites](#-prerequisites) |
| 3 | [Step 1 — Create the EKS Cluster](#step-1--create-the-eks-cluster) |
| 4 | [Step 2 — Connect to the Cluster](#step-2--connect-to-the-cluster) |
| 5 | [Step 3 — Create an RDS PostgreSQL Instance](#step-3--create-an-rds-postgresql-instance) |
| 6 | [Step 4 — Deploy the Application on EKS](#step-4--deploy-the-application-on-eks) |
| 7 | [Step 5 — Access the App (Port-Forward)](#step-5--access-the-app-port-forward) |
| 8 | [Step 6 — Set Up ALB Ingress Controller](#step-6--set-up-alb-ingress-controller) |
| 9 | [Step 7 — Create Ingress & Expose the App](#step-7--create-ingress--expose-the-app) |
| 10 | [Step 8 — Map a Custom Domain (Route 53)](#step-8--map-a-custom-domain-route-53) |
| 11 | [Author & Community](#-author--community) |

---

## 🏗 Architecture Overview

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│   React UI   │────▶│  Flask API   │────▶│  RDS PostgreSQL  │
│  (Frontend)  │     │  (Backend)   │     │   (Database)     │
└──────┬───────┘     └──────┬───────┘     └──────────────────┘
       │                    │
       └────────┬───────────┘
                ▼
        ┌───────────────┐
        │  AWS ALB via   │
        │  K8s Ingress   │
        └───────┬───────┘
                ▼
        ┌───────────────┐
        │  Route 53 DNS │
        └───────────────┘
```

| Layer | Technology | Details |
|-------|-----------|---------|
| **Frontend** | React | Served via Nginx on port `80` |
| **Backend** | Flask | REST API on port `8000` |
| **Database** | PostgreSQL (RDS) | Private subnet, port `5432` |
| **Orchestration** | EKS (Managed Node Group) | 2 nodes (`t3.medium`) |
| **Load Balancer** | AWS ALB | Provisioned by the AWS LB Controller |
| **DNS** | Route 53 | Custom domain alias to ALB |

---

## ✅ Prerequisites

Make sure the following tools are installed and configured **before** you begin:

| Tool | Purpose |
|------|---------|
| [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) | Interact with AWS services |
| [eksctl](https://eksctl.io/installation/) | Create & manage EKS clusters |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Manage Kubernetes resources |
| [Helm](https://helm.sh/docs/intro/install/) | Install Kubernetes charts |
| AWS Account Credentials | IAM user with admin permissions |

---

## Step 1 — Create the EKS Cluster

Create an EKS cluster with a **managed node group** (2 nodes, auto-scaling 1–3):

```bash
eksctl create cluster \
  --name Akhilesh-cluster \
  --region eu-west-1 \
  --version 1.31 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

> [!NOTE]
> This runs a CloudFormation stack that creates a **new VPC**, subnets, NAT Gateway, and the EKS cluster.

Verify the cluster was created:

```bash
aws eks list-clusters
```

---

## Step 2 — Connect to the Cluster

Configure `kubeconfig` so `kubectl` can talk to EKS:

```bash
export cluster_name=Akhilesh-cluster

aws eks update-kubeconfig --name $cluster_name --region eu-west-1

# Verify connection
kubectl config current-context
kubectl get nodes
kubectl get pods -A
```

---

## Step 3 — Create an RDS PostgreSQL Instance

The database lives in the **same VPC** as EKS, in **private subnets** only.

<details>
<summary><b>3.1 — Create a DB Subnet Group</b></summary>

```bash
# Get the VPC ID used by EKS
VPC_ID=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Find private subnets
PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" \
  --output text --region eu-west-1)

echo "Private subnets: $PRIVATE_SUBNET_IDS"

# Create subnet group (replace subnet IDs with your own)
aws rds create-db-subnet-group \
  --db-subnet-group-name akhilesh-postgres-private-subnet-group \
  --db-subnet-group-description "Private subnet group for PostgreSQL RDS" \
  --subnet-ids <SUBNET_1> <SUBNET_2> <SUBNET_3> \
  --region eu-west-1
```

</details>

<details>
<summary><b>3.2 — Create a Security Group for RDS</b></summary>

```bash
# Create security group
aws ec2 create-security-group \
  --group-name postgressg \
  --description "SG for RDS" \
  --vpc-id $VPC_ID \
  --region eu-west-1

# Get the new SG ID
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=postgressg" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text --region eu-west-1)

# Get the EKS cluster node SG
NODE_SG=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
  --query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)

# Allow EKS nodes → RDS on port 5432
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 5432 \
  --source-group $NODE_SG \
  --region eu-west-1
```

</details>

<details>
<summary><b>3.3 — Launch the RDS Instance</b></summary>

```bash
aws rds create-db-instance \
  --db-instance-identifier akhilesh-postgres \
  --db-instance-class db.t3.small \
  --engine postgres \
  --engine-version 15 \
  --allocated-storage 20 \
  --master-username postgresadmin \
  --master-user-password YourStrongPassword123! \
  --db-subnet-group-name akhilesh-postgres-private-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --multi-az \
  --storage-type gp2 \
  --region eu-west-1
```

Once created, note these values (you'll need them later):

| Parameter | Value |
|-----------|-------|
| `DB_HOST` | `akhilesh-postgres.<id>.eu-west-1.rds.amazonaws.com` *(from RDS console)* |
| `DB_NAME` | `postgres` |
| `USERNAME` | `postgresadmin` |
| `PASSWORD` | `YourStrongPassword123!` |

</details>

---

## Step 4 — Deploy the Application on EKS

### 4.1 — Clone the Repo

```bash
git clone https://github.com/NotHarshhaa/DevOps-Projects/DevOps-Project-36/3-tier-app-eks
cd 3-tier-app-eks/k8s
```

### 4.2 — Create the Namespace

```bash
kubectl apply -f namespace.yaml
```

### 4.3 — Create an ExternalName Service for RDS

This lets pods reach RDS via Kubernetes DNS (`postgres-db.3-tier-app-eks.svc.cluster.local`):

```bash
kubectl apply -f database-service.yaml
```

<details>
<summary>View the manifest — <code>database-service.yaml</code></summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: 3-tier-app-eks
  labels:
    service: database
spec:
  type: ExternalName
  externalName: akhilesh-postgres.<id>.eu-west-1.rds.amazonaws.com  # ← your RDS endpoint
  ports:
  - port: 5432
```

</details>

### 4.4 — Create Secrets & ConfigMap

```bash
# Generate base64 values
echo -n 'postgresadmin' | base64
echo -n 'YourStrongPassword123!' | base64

# Apply
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
```

### 4.5 — Run Database Migration Job

```bash
kubectl apply -f migration_job.yaml

# Monitor the job
kubectl get job -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
kubectl logs <migration-pod-name> -n 3-tier-app-eks
```

### 4.6 — Deploy Backend & Frontend

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml

# Verify everything is running
kubectl get deployment -n 3-tier-app-eks
kubectl get svc -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
```

---

## Step 5 — Access the App (Port-Forward)

Quick-test before setting up Ingress:

```bash
# Terminal 1 — Backend
kubectl port-forward -n 3-tier-app-eks svc/backend 8000:8000

# Terminal 2 — Frontend
kubectl port-forward -n 3-tier-app-eks svc/frontend 8080:80
```

| Service | URL |
|---------|-----|
| Backend API | `http://localhost:8000/api/topics` |
| Frontend UI | `http://localhost:8080` |

---

## Step 6 — Set Up ALB Ingress Controller

<details>
<summary><b>6.1 — Associate OIDC Provider</b></summary>

```bash
export cluster_name=Akhilesh-cluster

oidc_id=$(aws eks describe-cluster --name $cluster_name \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check if OIDC is already associated
aws iam list-open-id-connect-providers | grep $oidc_id

# If not, create it
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

</details>

<details>
<summary><b>6.2 — Create IAM Policy for the LB Controller</b></summary>

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

</details>

<details>
<summary><b>6.3 — Create IAM Service Account</b></summary>

```bash
eksctl create iamserviceaccount \
  --cluster=$cluster_name \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Verify
kubectl get serviceaccount -n kube-system | grep aws-load-balancer-controller
```

</details>

<details>
<summary><b>6.4 — Install the Controller via Helm</b></summary>

```bash
# Install CRDs
kubectl apply -k \
  "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

VPC_ID=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$cluster_name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$VPC_ID \
  --set region=eu-west-1
```

</details>

---

## Step 7 — Create Ingress & Expose the App

### 7.1 — Tag Public Subnets for ALB

> [!IMPORTANT]
> The ALB controller **requires** the tag `kubernetes.io/role/elb=1` on public subnets. Without it, ALB creation will fail.

```bash
VPC_ID=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)

# List public subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=map-public-ip-on-launch,Values=true" \
  --query "Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone}"

# Tag them (replace with your subnet IDs)
aws ec2 create-tags \
  --resources <SUBNET_1> <SUBNET_2> <SUBNET_3> \
  --tags Key=kubernetes.io/role/elb,Value=1
```

### 7.2 — Apply the Ingress Resource

```bash
kubectl apply -f ingress.yaml

# Verify
kubectl get ingress -n 3-tier-app-eks
kubectl describe ingress 3-tier-app-ingress -n 3-tier-app-eks
```

<details>
<summary>View the manifest — <code>ingress.yaml</code></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: 3-tier-app-ingress
  namespace: 3-tier-app-eks
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  ingressClassName: "alb"
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

</details>

Once the Ingress shows a **reconciled** status, an ALB is created. Copy the ALB DNS from the AWS console and open it in your browser.

---

## Step 8 — Map a Custom Domain (Route 53)

<details>
<summary><b>8.1 — Create a Hosted Zone</b></summary>

```bash
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config Comment="Public hosted zone for yourdomain.com"
```

Copy the **Name Servers** from the output and update them in your domain registrar (GoDaddy, Namecheap, etc.).

</details>

<details>
<summary><b>8.2 — Create an Alias Record → ALB</b></summary>

```bash
ALB_DNS=$(kubectl get ingress 3-tier-app-ingress -n 3-tier-app-eks \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name yourdomain.com \
  --query "HostedZones[0].Id" --output text | sed 's/\/hostedzone\///')

aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<ALB_HOSTED_ZONE_ID>",
          "DNSName": "'$ALB_DNS'",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

Wait a few minutes for DNS propagation, then visit `app.yourdomain.com` 🎉

</details>

---

## 🛠️ Troubleshooting

<details>
<summary><b>Can't connect to RDS from the cluster?</b></summary>

Run a debug pod to test connectivity:

```bash
kubectl run debug-pod --rm -it --image=postgres -- bash

# Inside the pod
PGPASSWORD=YourStrongPassword123! psql -h \
  postgres-db.3-tier-app-eks.svc.cluster.local -U postgresadmin -d postgres
```

**Common fix:** Ensure the RDS security group allows inbound on port `5432` from the EKS node security group.

</details>

<details>
<summary><b>ALB not getting created?</b></summary>

Check the controller logs:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Common fix:** Tag public subnets with `kubernetes.io/role/elb=1` (see [Step 7.1](#71--tag-public-subnets-for-alb)).

</details>

<details>
<summary><b>Ingress stuck? Force-delete it</b></summary>

```bash
kubectl patch ingress 3-tier-app-ingress -n 3-tier-app-eks \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

kubectl delete ingress 3-tier-app-ingress -n 3-tier-app-eks --grace-period=0 --force
```

</details>

---

## 🛠️ **Author & Community**

This project is crafted by [**Harshhaa**](https://github.com/NotHarshhaa) 💡.  
I'd love to hear your feedback! Feel free to share your thoughts.

---

### 📧 **Connect with me:**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/harshhaa-vardhan-reddy) [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/NotHarshhaa) [![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/prodevopsguy) [![Dev.to](https://img.shields.io/badge/Dev.to-0A0A0A?style=for-the-badge&logo=dev.to&logoColor=white)](https://dev.to/notharshhaa) [![Hashnode](https://img.shields.io/badge/Hashnode-2962FF?style=for-the-badge&logo=hashnode&logoColor=white)](https://hashnode.com/@prodevopsguy)

---

### 📢 **Stay Connected**

![Follow Me](https://imgur.com/2j7GSPs.png)
