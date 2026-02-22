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

![CloudFormation Output](https://miro.medium.com/v2/resize:fit:700/1*UxyId72AVYTLINj67y5b4w.png)

Verify the cluster was created:

```bash
aws eks list-clusters
```

![List Clusters](https://miro.medium.com/v2/resize:fit:700/1*nSlESN3rUF7DArDvuvlnGQ.png)

---

## Step 2 — Connect to the Cluster

Configure `kubeconfig` so `kubectl` can talk to EKS:

```bash
export cluster_name=Akhilesh-cluster

aws eks update-kubeconfig --name $cluster_name --region eu-west-1

# Verify connection
kubectl config current-context
```

![Kubeconfig Context](https://miro.medium.com/v2/resize:fit:700/1*gDUzKv5n3fcotKZvhavHNA.png)

```bash
kubectl get nodes
kubectl get pods -A
```

![Cluster Status](https://miro.medium.com/v2/resize:fit:700/1*L7W8OJm6VkRED79jmsCqIA.png)

---

## Step 3 — Create an RDS PostgreSQL Instance

The database lives in the **same VPC** as EKS, in **private subnets** only.

<details>
<summary><b>3.1 — Create a DB Subnet Group</b></summary>

```bash
# Get the VPC ID used by EKS
VPC_ID=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Create subnet group (replace subnet IDs with your own)
aws rds create-db-subnet-group \
  --db-subnet-group-name akhilesh-postgres-private-subnet-group \
  --db-subnet-group-description "Private subnet group for PostgreSQL RDS" \
  --subnet-ids <SUBNET_1> <SUBNET_2> <SUBNET_3> \
  --region eu-west-1
```

![Subnet Group Creation](https://miro.medium.com/v2/resize:fit:700/1*Xi42TFn0ibPh2vVN5Rpo3Q.png)

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
```

![Security Group Output](https://miro.medium.com/v2/resize:fit:700/1*pATG5bx1ZnNStfH6lEDIUw.png)

```bash
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

![Ingress Rule](https://miro.medium.com/v2/resize:fit:700/1*sczI87XY2YEOJMOiuQjOfQ.png)

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

![RDS Instance Creation](https://miro.medium.com/v2/resize:fit:700/1*Ali7XP8Gb1P-q4RzKrMs9A.png)

![RDS Status](https://miro.medium.com/v2/resize:fit:700/1*6wbcsMhKhv43p-6renC1cQ.png)

Once created, note these values (you'll need them later):

| Parameter | Value |
|-----------|-------|
| `DB_HOST` | `akhilesh-postgres.cveph9nmftjh.eu-west-1.rds.amazonaws.com` *(Find yours in RDS Console)* |
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
tree .
```

![Repo Structure](https://miro.medium.com/v2/resize:fit:700/1*IgAei8zoASURyRSWsIGNcw.png)

### 4.2 — Create the Namespace

```bash
kubectl apply -f namespace.yaml
```

![Namespace Created](https://miro.medium.com/v2/resize:fit:700/1*cgoHW2c2f4ga6uaAofPWAA.png)

### 4.3 — Create an ExternalName Service for RDS

This lets pods reach RDS via Kubernetes DNS (`postgres-db.3-tier-app-eks.svc.cluster.local`):

```bash
kubectl apply -f database-service.yaml
```

![Database Service](https://miro.medium.com/v2/resize:fit:700/1*bXl_GZch0YDUBgEL0fVxDw.png)

> [!TIP]
> You can test DNS resolution using:
> `kubectl run -it --rm --restart=Never dns-test --image=tutum/dnsutils -- dig postgres-db.3-tier-app-eks.svc.cluster.local`

![DNS Test](https://miro.medium.com/v2/resize:fit:700/1*och_4GdddRKMk843D0cWJQ.png)

### 4.4 — Create Secrets & ConfigMap

```bash
# Generate base64 values
echo -n 'postgresadmin' | base64
echo -n 'YourStrongPassword123!' | base64
```

![Base64 Encoding](https://miro.medium.com/v2/resize:fit:700/1*XDr5pzATja4yo9j5HVeRag.png)

```bash
# Apply
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
```

![ConfigMap/Secret Applied](https://miro.medium.com/v2/resize:fit:700/1*tEFK5HrgKtt8svaaC71WFA.png)

### 4.5 — Run Database Migration Job

```bash
kubectl apply -f migration_job.yaml

# Monitor the job
kubectl get job -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
kubectl logs <migration-pod-name> -n 3-tier-app-eks
```

![Migration Job](https://miro.medium.com/v2/resize:fit:700/1*BRAgSEqM3MpMdhzRILhUnw.png)

### 4.6 — Deploy Backend & Frontend

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
```

![Deployments Applied](https://miro.medium.com/v2/resize:fit:700/1*-tMqfKLQta4X4poJC1nSNA.png)

```bash
# Verify everything is running
kubectl get deployment -n 3-tier-app-eks
kubectl get svc -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
```

![Pods & Services](https://miro.medium.com/v2/resize:fit:700/1*7EOcnII-zIyQFSwcsk5q1g.png)

---

## Step 5 — Access the App (Port-Forward)

Quick-test before setting up Ingress:

```bash
# Terminal 1 — Backend
kubectl port-forward -n 3-tier-app-eks svc/backend 8000:8000

# Terminal 2 — Frontend
kubectl port-forward -n 3-tier-app-eks svc/frontend 8080:80
```

![Port Forwarding](https://miro.medium.com/v2/resize:fit:700/1*_AXwf7qnfTBKpxhFtHb7mQ.png)

| Service | URL |
|---------|-----|
| Backend API | `http://localhost:8000/api/topics` |
| Frontend UI | `http://localhost:8080` |

![API Output](https://miro.medium.com/v2/resize:fit:700/1*cPCOlZjFHoaiMBAukB3YCA.png)

![Frontend UI](https://miro.medium.com/v2/resize:fit:700/1*WYC1cw1fPxwEvqU72N3e9Q.png)

---

## Step 6 — Set Up ALB Ingress Controller

<details>
<summary><b>6.1 — Associate OIDC Provider</b></summary>

```bash
export cluster_name=Akhilesh-cluster

oidc_id=$(aws eks describe-cluster --name $cluster_name \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# If not, create it
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

![OIDC Created](https://miro.medium.com/v2/resize:fit:700/1*f8HmizOKvNog9nMtBsP5yQ.png)

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
```

![IAM Service Account](https://miro.medium.com/v2/resize:fit:700/1*5kCLDmn1AX2qmaivvME9wg.png)

</details>

---

## Step 7 — Create Ingress & Expose the App

### 7.1 — Tag Public Subnets for ALB

> [!IMPORTANT]
> The ALB controller **requires** the tag `kubernetes.io/role/elb=1` on public subnets.

```bash
# Tag them (replace with your subnet IDs from Step 3)
aws ec2 create-tags \
  --resources <SUBNET_1> <SUBNET_2> <SUBNET_3> \
  --tags Key=kubernetes.io/role/elb,Value=1
```

![Subnet Tagging Success](https://miro.medium.com/v2/resize:fit:700/1*74_D4X1fgXCsqscMF6mqyw.png)

### 7.2 — Apply the Ingress Resource

```bash
kubectl apply -f ingress.yaml

# Verify
kubectl get ingress -n 3-tier-app-eks
```

![Ingress Reconciled](https://miro.medium.com/v2/resize:fit:700/1*Iy8Sy7pyumUzy9Tw7Scy-Q.png)

Once the Ingress shows a **reconciled** status, an ALB is created. 

![ALB Provisioned](https://miro.medium.com/v2/resize:fit:700/1*edYV1dfcXHtEpuekvbfNcw.png)

![App Loading via ALB](https://miro.medium.com/v2/resize:fit:700/1*AhXAPTww7KqC09mO15rS8Q.png)

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

![Route53 Hosted Zone](https://miro.medium.com/v2/resize:fit:700/1*wPGs8npRZ-TCYFBDOjNQkw.png)

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

![Connectivity Issue Example](https://miro.medium.com/v2/resize:fit:700/1*j0Zn5lhcV8F1c94_vYIjKQ.png)

**Common fix:** Ensure the RDS security group allows inbound on port `5432` from the EKS node security group. In some cases, opening all IPs for testing might confirm if it's a security group issue.

![All IPs Test](https://miro.medium.com/v2/resize:fit:700/1*ISkiSMNvqVbcsjnqcJzR3Q.png)

</details>

<details>
<summary><b>ALB not getting created?</b></summary>

Check the controller logs:
`kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`

If you see tagging errors like below, you must tag your subnets correctly.

![Subnet Tagging Error](https://miro.medium.com/v2/resize:fit:700/1*MMV1QT-B2MU1FRxJDNMBLQ.png)

</details>

<details>
<summary><b>Ingress stuck? Force-delete it</b></summary>

```bash
kubectl patch ingress 3-tier-app-ingress -n 3-tier-app-eks \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

kubectl delete ingress 3-tier-app-ingress -n 3-tier-app-eks --grace-period=0 --force
```

</details>

