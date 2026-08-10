# Medicare CI/CD Pipeline

An automated Continuous Integration and Continuous Deployment (CI/CD) pipeline built with **Jenkins**, **Docker**, **AWS EC2**, and **GitHub Webhooks**. This project automates the workflow from code commit on GitHub to automated container deployment on an AWS EC2 host.

---

## 🛠 Tech Stack & Tools

* **Version Control:** GitHub
* **CI/CD Tool:** Jenkins (Running via Docker)
* **Containerization:** Docker
* **Cloud Infrastructure:** AWS EC2 (Ubuntu 26.04 LTS)
* **Virtualization Host:** Oracle VirtualBox (Ubuntu Local Environment)
* **Protocol/Access:** SSH / MobaXterm

---

## 📐 Architecture & Workflow
[ GitHub Repo ] ---> (Webhook Trigger) ---> [ Jenkins CI/CD Pipeline ]
|
v
[ Docker Build & Deploy ]
|
v
[ AWS EC2 Instance ]
1. Developer pushes code updates to the `Medicare-CICD-Pipeline` GitHub repository.
2. GitHub sends an automated POST event trigger via Webhook to the Jenkins server.
3. Jenkins pulls the latest source code using the defined `Jenkinsfile`.
4. Jenkins builds the application Docker container image and deploys it on the target host server.

---

## 🚀 Key Configurations & Setup Highlights

### 1. EC2 & Environment Setup
* Configured an AWS EC2 Ubuntu instance hosting the primary deployment environment.
* Configured VirtualBox swap space allocation to optimize host VM memory performance.
* Set up root and non-root user privileges for Docker execution without permissions issues.

### 2. Jenkins Containerization
* Deployed Jenkins inside Docker on port `8080` with SCM access configured on port `50000`.
* Persisted job data and configurations using Docker volume mounts (`jenkins_home`).

### 3. GitHub Webhook Integration
* Configured GitHub repository webhooks targeting:
  `http://<EC2-IP>:8080/github-webhook/`
* Enabled **GitHub hook trigger for GITScm polling** inside the job pipeline configurations.

---

## ⚠️ Important Troubleshooting & Security Fixes

### Resolving HTTP 403 / "No Valid Crumb" Webhook Errors
During initial integration, GitHub webhooks failed with an `HTTP 403 Forbidden` response (`No valid crumb was included in the request`). This was resolved by:

1. **Ensuring Correct Endpoint:** Updating the webhook Payload URL to use `/github-webhook/` instead of direct job endpoints.
2. **Verifying Active Plugins:** Ensuring the **GitHub Plugin** and **GitHub API Plugin** are enabled in Jenkins.
3. **CSRF Bypassing for Automated Triggers:** Launching the Jenkins Docker container with explicit Java options to disable strict CSRF crumb checks for external API triggers:

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  --name jenkins \
  -v jenkins_home:/var/jenkins_home \
  -e JAVA_OPTS="-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:lts