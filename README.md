#  Developed a cloud-native e-commerce platform composed of 11 microservices deployed on Kubernetes, featuring a complete DevSecOps pipeline with GitOps, Jenkins, Docker, SonarQube for code quality, and observability using Prometheus and Grafana.

## This project Forked from the https://github.com/GoogleCloudPlatform/microservices-demo (Org source)

## For more projects, check out  
[https://github.com/elsonpulikkan96](https://github.com/elsonpulikkan96)


**Online Boutique** is a cloud-first microservices demo application.  The application is a
web-based e-commerce app where users can browse items, add them to the cart, and purchase them.


## Architecture

[![Architecture of
microservices](/docs/img/architecture-diagram.png)](https://github.com/elsonpulikkan96)


| Service                                              | Language      | Description                                                                                                                       |
| ---------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [frontend](/src/frontend)                           | Go            | Exposes an HTTP server to serve the website. Does not require signup/login and generates session IDs for all users automatically. |
| [productcatalogservice](/src/productcatalogservice) | Go            | Provides the list of products from a JSON file and ability to search products and get individual products.                        |
| [shippingservice](/src/shippingservice)             | Go            | Gives shipping cost estimates based on the shopping cart. Ships items to the given address (mock)                                 |
| [checkoutservice](/src/checkoutservice)             | Go            | Retrieves user cart, prepares order and orchestrates the payment, shipping and the email notification.                            |
| [cartservice](/src/cartservice)                     | C#            | Stores the items in the user's shopping cart in Redis and retrieves it.                                                           |
| [currencyservice](/src/currencyservice)             | Node.js       | Converts one money amount to another currency. Uses real values fetched from European Central Bank. It's the highest QPS service. |
| [paymentservice](/src/paymentservice)               | Node.js       | Charges the given credit card info (mock) with the given amount and returns a transaction ID.                                     |
| [emailservice](/src/emailservice)                   | Python        | Sends users an order confirmation email (mock).                                                                                   |
| [recommendationservice](/src/recommendationservice) | Python        | Recommends other products based on what's given in the cart.                                                                      |
| [loadgenerator](/src/loadgenerator)                 | Python/Locust | Continuously sends requests imitating realistic user shopping flows to the frontend.                                              |
| [adservice](/src/adservice)                         | Java          | Provides text ads based on given context words.                                                                                   |

## Screenshots

| Home Page                                                                                                         | Checkout Screen                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [![Screenshot of store homepage](/docs/img/online-boutique-frontend-1.png)](/docs/img/online-boutique-frontend-1.png) | [![Screenshot of checkout screen](/docs/img/online-boutique-frontend-2.png)](/docs/img/online-boutique-frontend-2.png) |


## Table of Contents

- [Prerequisites](#prerequisites)
- [System Update & Common Packages](#system-update--common-packages)
- [Java](#java)
- [Jenkins](#jenkins)
- [Docker](#docker)
- [Trivy](#trivy-vulnerability-scanner)
- [Jenkins Plugins to Install](#jenkins-plugins-to-install)
- [Jenkins Credentials to Store](#jenkins-credentials-to-store)
- [Jenkins Tools Configuration](#jenkins-tools-configuration)
- [Jenkins System Configuration](#jenkins-system-configuration)
- [EKS ALB Ingress Kubernetes Setup Guide](#eks-alb-ingress-kubernetes-setup-guide)
- [Monitor Kubernetes with Prometheus](#monitor-kubernetes-with-prometheus)
- [Installing Argo CD](#installing-argo-cd)
- [Notes and Recommendations](#notes-and-recommendations)

---

## Prerequisites

- Instance Type :- c5.xlarge
This guide assumes an Ubuntu/Debian-like environment and sudo privileges.

---

## Ports to Enable in Security Group

| Service         | Port  |
|-----------------|-------|
| HTTP            | 80    |
| HTTPS           | 443   |
| SSH             | 22    |
| Jenkins         |       |
| SonarQube       |       |

## System Update & Common Packages

```bash
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```
Reload bash completion if needed:
```bash
source /etc/bash_completion
```

**Install latest Git:**
```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git -y
```

---

## Java

Install OpenJDK (choose 17 or 21 depending on your needs):

```bash
# OpenJDK 17
sudo apt install -y openjdk-17-jdk

# OR OpenJDK 21
sudo apt install -y openjdk-21-jdk
```
Verify:
```bash
java --version
```

---

## Jenkins

Official docs: https://www.jenkins.io/doc/book/installing/linux/

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
Initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open: http://your-server-ip:8080

**Note:** Jenkins requires a compatible Java runtime. Check the Jenkins documentation for supported Java versions.

---

## Docker

Official docs: https://docs.docker.com/engine/install/ubuntu/

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group (log out / in or newgrp to apply)
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
If Jenkins needs Docker access:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Check Docker status:
```bash
sudo systemctl status docker
```

---

## Trivy (Vulnerability Scanner)

Docs: https://trivy.dev/v0.65/getting-started/installation/

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy


trivy --version
```

## Python Package Installation in the Amazon Ubuntu Ami Image [ Preinstalled 3.12 ]

```bash
# 1. Update package list
sudo apt update

# 2. Install required dependencies for adding a new Python version
sudo apt install -y software-properties-common

# 3. Add the deadsnakes PPA (Personal Package Archive) to get newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# 4. Install the specific version of Python you want
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev

curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.10


ls /usr/bin/python3*

sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 2
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1


sudo update-alternatives --config python3

```
## Choose python3.10 for Jenkins pipeline.

## If your using the Plan VM
```bash
sudo apt-get update
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev
```
---

## Jenkins Plugins to Install

- Eclipse Temurin installer Plugin
- NodeJS
- Email Extension Plugin
- OWASP Dependency-Check Plugin
- Pipeline: Stage View Plugin
- SonarQube Scanner for Jenkins
- Go
- .NET SDK Support
- Python Pyenv



---
## SonarQube Docker Container Run for Analysis

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:25.9.0.112764-community
```

---

## Jenkins Credentials to Store

| Purpose       | ID            | Type          | Notes                               |
|---------------|---------------|---------------|-------------------------------------|
| Email         | mail-cred     | Username/app password |                                  |
| SonarQube     | sonar-token   | Secret text   | From SonarQube application         |
| Docker Hub    | docker-cred   | Secret text   | From your Docker Hub profile       |

Webhook example:  
`http://<jenkins-ip>:8080/sonarqube-webhook/`

---

## Jenkins Tools Configuration

- JDK [jdk17 , jdk21 ]
- SonarQube Scanner installations [sonar-scanner]
- Node [ node16 , node20 ]
- Dependency-Check installations [dp-check]
- Go [ go1.25 ]
- .NET SDK installations [ dotnet9 ]
  | .NET 9.0 - Status Unknown (end of support: 2026-11-10) |
  |9.0.9, released 2025-09-09 |
  |9.0.305|
  |linux-x64 (Linux - x64)|
---

## Jenkins System Configuration

**SonarQube servers:**   
- Name: sonar-server  
- URL: http://sonar-ip-address:9000  
- Credentials: Add from Jenkins credentials

**Extended E-mail Notification:**
- SMTP server: smtp.gmail.com
- SMTP Port: 465
- Use SSL
- Default user e-mail suffix: @gmail.com

**E-mail Notification:**
- SMTP server: smtp.gmail.com
- Default user e-mail suffix: @gmail.com
- Use SMTP Authentication: Yes
- User Name: example@gmail.com
- Password: Use credentials
- Use TLS: Yes
- SMTP Port: 587
- Reply-To Address: example@gmail.com

---
# Now See the configuration pipeline of the Jenkins



## EKS ALB Ingress Kubernetes Setup Guide
# EKS cluster setup and  ALB Ingress Kubernetes Setup Guide

This guide covers the installation and setup for AWS CLI, `kubectl`, `eksctl`, and `helm`, and creating/configuring an EKS cluster with AWS Load Balancer Controller.

---

## 1. AWS CLI Installation

Refer: [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```bash
sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## 2. kubectl Installation

Refer: [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl bash-completion

# Enable kubectl auto-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 3. eksctl Installation

Refer: [eksctl Installation Guide](https://eksctl.io/installation/)

```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

# Install bash completion
sudo apt-get install -y bash-completion

# Enable eksctl auto-completion
echo 'source <(eksctl completion bash)' >> ~/.bashrc
echo 'alias e=eksctl' >> ~/.bashrc
echo 'complete -F __start_eksctl e' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 4. Helm Installation

Refer: [Helm Installation Guide](https://helm.sh/docs/intro/install/)

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm bash-completion

# Enable Helm auto-completion
echo 'source <(helm completion bash)' >> ~/.bashrc
echo 'alias h=helm' >> ~/.bashrc
echo 'complete -F __start_helm h' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 5. AWS CLI Configuration

```bash
aws configure
aws configure list
```


---

## 6. Create EKS Cluster and Nodegroup (Try-This)

```bash
eksctl create cluster \
  --name store-elsondevops-cloud \
  --region us-east-1 \
  --version 1.34 \
  --without-nodegroup

eksctl create nodegroup \
  --cluster store-elsondevops-cloud \
  --name worker \
  --nodes 3 \
  --nodes-min 3  \
  --nodes-max 4 \
  --node-type t3.medium
```

---

## 7. Update kubeconfig

```bash
aws eks update-kubeconfig --name store-elsondevops-cloud --region us-east-1
```

---

## 8. Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster store-elsondevops-cloud --approve
```

---

## 9. Create IAM Policy for AWS Load Balancer Controller

New policy link: [AWS EKS LBC Policy](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html)

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

## 10. Create IAM Service Account

Replace `<ACCOUNT_ID>` with your AWS account ID.

```bash
eksctl create iamserviceaccount \
  --cluster=store-elsondevops-cloud \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::739275449845:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

---

## 11. Install AWS Load Balancer Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=store-elsondevops-cloud \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --version 1.14.1
```

**Optional:** List available versions:
```bash
helm search repo eks/aws-load-balancer-controller --versions
helm list -A
```

**Verify installation:**
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Monitor Kubernetes with Prometheus

**Install Node Exporter using Helm:**

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus-community
```

```bash
kubectl create namespace prometheus
```


```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

```bash
kubectl get pods -n prometheus
```

```bash
kubectl get svc -n prometheus
```

## Edit Prometheus Service
```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```

## Edit Grafana Service
```bash
kubectl edit svc stable-grafana -n prometheus
```

```bash
kubectl get svc -n prometheus
```

## Grafana Login Details
|                                               |     |                                                                                                                        
| ---------------------------------------------------- | ------------- | 
| UserName | admin |
| Password  | prom-operator |

|                                               |     |                        
| ---------------------------------------------------- | ------------- | 
| Kubernetes Monitoring Dashboard | 12740 |
| Node Exporter | 1860 |
| Kubernetes / Views / Namespace | 15758 |

---
---

## Installing Argo CD on the eks cluster

  - Docs: https://www.eksworkshop.com/docs/automation/gitops/argocd/access_argocd
  - Docs: https://github.com/argoproj/argo-helm

# Argocd installation via helm chart

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

```bash
kubectl create namespace argocd 
helm install argocd argo/argo-cd --namespace argocd
kubectl get all -n argocd 
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}' 
```
# Another way to get the loadbalancer of the argocd alb url

```bash
sudo apt install jq -y

kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'
```

Username: admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
---
Password: encrypted-password
---

##  Delete EKS Cluster (Cleanup) finally u done a project 
 - For more conents reach out : https://github.com/elsonpulikkan96/

```bash
eksctl delete cluster --name store-elsondevops-cloud --region us-east-1
```

## Notes and Recommendations

- Replace `<VERSION>`, `<your-server-ip>`, and other placeholders with specific values for your setup.
- Prefer pinned versions for production environments rather than "latest".
- Consult each project's official documentation for the most up-to
