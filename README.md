# 🐍 Jenkins CI/CD Pipeline — Python App to AWS ECR

Automated pipeline that builds a Docker image and pushes it to AWS ECR on every `git push`.

---

## 🏗️ Pipeline Flow
```
GitHub Push → Webhook →  Jenkins Agent (EC2) → Docker Build → Push to ECR
```

---

## 🖥️ Infrastructure

> ✅ **Recommended:** Run Jenkins controller and agent on **separate EC2 instances**.
> The controller only orchestrates — all actual build work runs on the agent.

| Component | Instance | Role |
|---|---|---|
| Jenkins Controller | EC2 #1 | Jenkins UI, job scheduling, webhook receiver |
| Jenkins Agent | EC2 #2 | Runs pipeline — builds image, pushes to ECR |
| AWS ECR | AWS Managed | Stores Docker images |
| GitHub | Git Remote | Source code + webhook trigger |

---

## 📋 Prerequisites

### AWS
- [ ] Two EC2 instances launched (Amazon Linux 2023 recommended)
- [ ] Security group on **Controller EC2**: port `8080` (Jenkins UI) and `22` (SSH) open
- [ ] ECR repository created: `python-app-jenkins`
- [ ] IAM Role with ECR permissions attached to **Agent EC2**

### Controller EC2 — Install Jenkins
```bash
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install -y jenkins java-17-amazon-corretto
sudo systemctl start jenkins && sudo systemctl enable jenkins
```
Access Jenkins at: `http://<CONTROLLER_IP>:8080`

### Agent EC2 — Install Required Tools
```bash
sudo yum update -y
sudo yum install -y git java-17-amazon-corretto

# Docker
sudo yum install -y docker
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Verify
git --version && docker --version && aws --version && java -version
```

---

## ⚙️ Setup Steps

### 1. Connect Agent to Jenkins Controller
```
Jenkins UI → Manage Jenkins → Nodes → New Node
→ Node name: ec2-agent
→ Type: Permanent Agent
→ Remote root directory: /home/ec2-user
→ Label: ec2-agent
→ Launch method: Launch agents via SSH
→ Host: <AGENT_EC2_PRIVATE_IP>
→ Credentials: Add SSH private key of agent EC2
→ Save
```
Verify the agent shows **ONLINE** in `Manage Jenkins → Nodes`.

### 2. IAM Role for Agent EC2
Attach a role with `AmazonEC2ContainerRegistryPowerUser` to the **agent EC2**:
```
AWS Console → EC2 → Agent Instance → Actions → Security → Modify IAM Role
```

### 3. Jenkins Credentials
Never hardcode AWS details in Jenkinsfile. Store as secrets:
```
Manage Jenkins → Credentials → Global → Add Credentials → Secret text
```
| ID | Value |
|---|---|
| `AWS_ACCOUNT_ID` | your 12-digit AWS account ID |
| `AWS_REGION_NAME` | `us-east-1` |

### 4. Jenkinsfile


### 5. GitHub Webhook (Auto-trigger on push)
```
GitHub Repo → Settings → Webhooks → Add webhook
  Payload URL:  http://<CONTROLLER_IP>:8080/github-webhook/
  Content type: application/json
  Event:        Just the push event → Add webhook
```
Enable trigger in Jenkins job:
```
Job → Configure → Build Triggers
→ ✅ GitHub hook trigger for GITScm polling → Save
```
Required plugins (must be **enabled** — blue toggle):
```
Manage Jenkins → Plugins → Installed → search "github"
→ GitHub plugin ✅
→ GitHub API Plugin ✅
```
If a plugin is stuck disabled:
```bash
cd /var/lib/jenkins/plugins
sudo rm -f github.jpi.disabled github-api.jpi.disabled
sudo systemctl restart jenkins
```

---

## 🐛 Common Issues & Fixes

| Error | Fix |
|---|---|
| Agent offline | Check `/tmp` disk: `df -h /tmp` — clear if full |
| Docker permission denied | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| ECR login fails | Verify IAM role is attached to **agent** EC2 |
| `ERROR: AWS_ACCOUNT_ID` | Credential ID must exactly match Jenkins (case-sensitive) |
| Webhook delivers ✅ but no build | Enable "GitHub hook trigger" in Job → Configure → Build Triggers |
| Plugin toggle unclickable | Remove `.disabled` file via SSH (see Step 5 above) |
