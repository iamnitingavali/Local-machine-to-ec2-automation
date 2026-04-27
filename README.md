# 🚀 Local Mnachine to EC2 CI/CD Pipeline with Jenkins & PM2

A production-ready **Continuous Integration / Continuous Deployment** pipeline that automatically deploys a Node.js Express app to an AWS EC2 instance using Jenkins, SSH, and PM2 — every time you push to GitHub.

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             CI/CD PIPELINE FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────┐         ┌──────────────────┐        ┌──────────────────┐
  │  LOCAL MACHINE   │         │   GITHUB REPO    │        │  AWS EC2 SERVER  │
  │ (Local Machine)  │         │  (Central Vault) │        │(Live Production) │
  └────────┬─────────┘         └────────┬─────────┘        └─────────┬────────┘
           │                            │                            │
           │  1. git push origin main   │                            │
           │ ─────────────────────────► │                            │
           │                            │                            │
           │                            │  2. Webhook triggers       │
           │                            │     Jenkins                │
           │                            │ ─────────────────────────► │
           │                            │                            │
           │                            │       ┌────────────────────┤
           │                            │       │  JENKINS PIPELINE  │
           │                            │       │  ┌──────────────┐  │
           │                            │       │  │ Stage 1:     │  │
           │                            │       │  │ Clone Repo   │  │
           │                            │       │  └──────┬───────┘  │
           │                            │       │         │          │
           │                            │       │  ┌──────▼───────┐  │
           │                            │       │  │ Stage 2:     │  │
           │                            │       │  │ Upload (SCP) │  │
           │                            │       │  └──────┬───────┘  │
           │                            │       │         │          │
           │                            │       │  ┌──────▼───────┐  │
           │                            │       │  │ Stage 3:     │  │
           │                            │       │  │ npm install &│  │
           │                            │       │  │ pm2 restart  │  │
           │                            │       │  └──────────────┘  │
           │                            │       └────────────────────┤
           │                            │                            │
           │                            │          3. App is LIVE    │
           │                            │ ◄───────────────────────── │
           │                            │                            │

   GOLDEN RULE:  Local Machine  ──►  GitHub  ──►  EC2 (via Jenkins)
```

---

## 🗂️ Project Structure

```
node-app/
├── app.js          # Express server — the heart of the application
├── package.json    # Project metadata and dependencies
└── Jenkinsfile     # Declarative pipeline — the CI/CD brain
```

---

## ⚙️ Tech Stack

| Layer | Technology | Role |
|---|---|---|
| **App** | Node.js + Express | HTTP server |
| **Process Manager** | PM2 | Keeps app alive, auto-restarts |
| **Version Control** | Git + GitHub | Source of truth |
| **CI/CD Engine** | Jenkins | Automation server |
| **Cloud Server** | AWS EC2 (Ubuntu) | Production host |
| **Secure Transfer** | SSH + SCP | Jenkins ↔ EC2 communication |
| **Dev Environment** | Local Machine | Local development terminal |

---

## 🔄 Pipeline Stages (Jenkinsfile Explained)

The `Jenkinsfile` defines **3 automated stages** that run on every push:

### Stage 1 — `Clone Repository`
Jenkins clones the latest code from the `main` branch of the GitHub repo directly into the Jenkins workspace.

```groovy
stage('Clone Repository') {
    steps {
        git branch: "${BRANCH}", url: "${REPO_URL}"
    }
}
```

### Stage 2 — `Upload Files to EC2`
Using SSH credentials stored securely in Jenkins, the pipeline SSHs into the EC2 instance, creates the app directory if it doesn't exist, then transfers all project files via `scp`.

```groovy
stage('Upload Files to EC2') {
    steps {
        sshagent([SSH_CREDENTIAL]) {
            sh """
                ssh ... 'mkdir -p ${REMOTE_PATH}'
                scp -r * ${REMOTE_USER}@${SERVER_IP}:${REMOTE_PATH}/
            """
        }
    }
}
```

### Stage 3 — `Install Dependencies & Start App`
Still over SSH, Jenkins runs `npm install` on the server, then either starts the app fresh with PM2 or restarts the existing process — achieving **zero-downtime deployments**.

```groovy
stage('Install Dependencies & Start App') {
    steps {
        sshagent([SSH_CREDENTIAL]) {
            sh """
                ssh ... '
                    cd ${REMOTE_PATH} &&
                    npm install &&
                    pm2 start app.js --name node-app || pm2 restart node-app
                '
            """
        }
    }
}
```

---

## 🌍 Environment Variables

These are configured inside the `Jenkinsfile` under the `environment` block:

| Variable | Description |
|---|---|
| `SERVER_IP` | Public IP of your EC2 instance |
| `SSH_CREDENTIAL` | Jenkins credential ID for your EC2 SSH key |
| `REPO_URL` | GitHub repository URL |
| `BRANCH` | Git branch to deploy (`main`) |
| `REMOTE_USER` | EC2 login user (e.g., `ubuntu`) |
| `REMOTE_PATH` | Directory on EC2 where app files live |

---

## 📋 Prerequisites

Before this pipeline works, ensure the following are set up:

- [ ] **Jenkins** installed and running (on EC2 or a separate server)
- [ ] **Jenkins Plugins**: `Git Plugin`, `SSH Agent Plugin`
- [ ] **SSH Credentials** added to Jenkins (`node-app-key`)
- [ ] **Node.js** installed on the EC2 instance
- [ ] **PM2** installed globally on EC2: `npm install -g pm2`
- [ ] **GitHub Webhook** configured to trigger Jenkins on every push
- [ ] EC2 **Security Group** allows inbound traffic on port `3000`

---

## 🚀 Getting Started

### 1. Clone the repo locally

```bash
git clone https://github.com/iamnitingavali/Local-machine-to-ec2-automation.git
cd Local-machine-to-ec2-automation
```

### 2. Run locally (optional test)

```bash
npm install
npm start
# Visit http://localhost:3000
```

### 3. Push changes to trigger the pipeline

```bash
git add .
git commit -m "your change description"
git push origin main
# Jenkins picks this up automatically via webhook ✅
```

---

## 🔁 How the Deployment Loop Works

```
You write code
      │
      ▼
git push origin main
      │
      ▼
GitHub receives the push
      │
      ▼  (webhook fires)
Jenkins starts the pipeline
      │
      ├── Stage 1: Pulls latest code from GitHub
      │
      ├── Stage 2: Copies files to EC2 over SSH/SCP
      │
      └── Stage 3: Runs npm install + pm2 restart on EC2
                          │
                          ▼
               App is live at http://<EC2-IP>:3000 🎉
```

---

## ✅ Post-Build Notifications

The pipeline reports its result after every run:

```
✅ Application deployed successfully!   ← on success
❌ Deployment failed.                   ← on failure
```

Check the **Jenkins Build Console Output** for full logs on any run.

---

## 🛡️ Security Notes

- The EC2 `.pem` private key is **never** stored in the repo. It lives in Jenkins Credentials Manager.
- `StrictHostKeyChecking=no` is used for automation — consider adding the EC2 host key to `known_hosts` in production.
- Restrict EC2 Security Group inbound rules to only necessary IPs for port `3000`.

---

## 📌 The Golden Rule

> **Local Machine → GitHub → EC2**
>
> Never SSH directly to the server and edit code manually.  
> Always push to GitHub and let Jenkins handle the rest.  
> This ensures your code is always version-controlled, backed up, and your server always matches the repo.

---

## 👤 Author

**Nitin Gavali**  
GitHub: [@iamnitingavali](https://github.com/iamnitingavali)

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

