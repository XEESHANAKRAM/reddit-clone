
# 🚀 Jenkins, Docker, SonarQube & EKS Setup using Terraform

This guide helps you deploy a CI/CD environment on AWS using Terraform. It includes:

- Jenkins
- Docker
- SonarQube
- Trivy
- Kubernetes (EKS)
- ArgoCD
- Prometheus & Grafana for Monitoring

---

## [A] Create EC2 Instance with Terraform

### 1. `main.tf`

```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0287a05f0ef0e9d9a"  # Change for your region
  instance_type          = "t2.large"
  key_name               = "Linux-VM-Key7"
  vpc_security_group_ids = [aws_security_group.Jenkins-VM-SG.id]
  user_data              = templatefile("./install.sh", {})
  tags = {
    Name = "Jenkins-SonarQube"
  }
  root_block_device {
    volume_size = 40
  }
}

resource "aws_security_group" "Jenkins-VM-SG" {
  name        = "Jenkins-VM-SG"
  description = "Allow inbound traffic"
  ingress = [
    for port in [22, 80, 443, 8080, 9000, 3000] : {
      description      = "inbound"
      from_port        = port
      to_port          = port
      protocol         = "tcp"
      cidr_blocks      = ["0.0.0.0/0"]
    }
  ]
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "Jenkins-VM-SG"
  }
}
```

### 2. `provider.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

### 3. `install.sh`

```bash
#!/bin/bash
sudo apt update -y

# Install JDK 17
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins

# Install Docker & SonarQube
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
newgrp docker
sudo chmod 777 /var/run/docker.sock
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

### 4. Run Terraform

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

---

### References
EKS Workshop: ArgoCD Install

#AWS EKS GitHub Issue

#ArgoCD CLI Installation

### [B] 🚀 Create AWS EKS Cluster
# 1. Install kubectl on Jenkins Server
```sudo apt update
sudo apt install curl
curl -LO https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

# 2. Install AWS CLI

```curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
# 3. Install eksctl

```curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
cd /tmp
sudo mv /tmp/eksctl /bin
eksctl version
```
# 4. Create EKS Cluster

```eksctl create cluster --name virtualtechbox-cluster \
--region ap-south-1 \
--node-type t2.small \
--nodes 3
```
# 5. Verify EKS Cluster

```kubectl get nodes
```
### [C] 📊 Monitoring Setup: Helm, Prometheus & Grafana
# 1. Install Helm

```curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```
# 2. Add Helm Repositories

```helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus
```
# 3. Install Prometheus Stack

```helm install stable prometheus-community/kube-prometheus-stack -n prometheus
kubectl get pods -n prometheus
kubectl get svc -n prometheus
```
# 4. Expose Prometheus & Grafana

```kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```
### Change to LoadBalancer, set port/targetPort to 9090

```kubectl edit svc stable-grafana -n prometheus
```
### Change to LoadBalancer

```kubectl get svc -n prometheus
```
# 5. Get Grafana Admin Password

```kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
# 6. Import Dashboards in Grafana
```Dashboard ID: 15760
Dashboard ID: 12740
```

# [D] 🎯 ArgoCD Installation & EKS Integration
# 1. Create Namespace

```kubectl create namespace argocd
```
# 2. Install ArgoCD

```kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
# 3. Verify ArgoCD Pods

```kubectl get pods -n argocd
```
# 4. Install ArgoCD CLI

```sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```
# 5. Expose ArgoCD to Internet

```kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd
```
# 6. Get ArgoCD Admin Password

```kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
echo WXVpLUg2LWxoWjRkSHFmSA== | base64 --decode
```
# 7. Login to ArgoCD via CLI

```argocd login <ARGOCD_LB_DNS> --username admin
```
# 8. List Available Clusters

```argocd cluster list
kubectl config get-contexts
```
# 9. Add EKS Cluster to ArgoCD

```argocd cluster add i-08b9d0ff0409f48e7@virtualtechbox-cluster.ap-south-1.eksctl.io --name virtualtechbox-eks-cluster
```
# [E] 🔁 CI/CD Pipeline Test

```git config --global user.name "XEESHANAKRAM"
git config --global user.email "jamzeeshanakram@gmail.com"
git clone https://github.com/XEESHANAKRAM/reddit-clone.git
```
[F] 🧹 Cleanup Steps
# 1. Delete Prometheus & ArgoCD Namespaces

```kubectl delete namespace prometheus
kubectl delete namespace argocd
```
# 2. Delete EKS Cluster

```eksctl delete cluster virtualtechbox-cluster --region ap-south-1
```
# 3. Destroy Terraform Infrastructure

```terraform destroy
```
