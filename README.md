# Jenkins on Azure VM (Ubuntu) — Installation & Configuration Guide

A clean, step‑by‑step guide to launch a Linux VM on Azure, install Jenkins with Java 21, enable Docker integration, and run a simple pipeline.

---

## 1) Overview

* **OS:** Ubuntu LTS (22.04/24.04)
* **VM name:** `jenkins`
* **Auth:** SSH key (`jenkins.pem`)
* **NSG inbound:** TCP **8080** (Jenkins UI), **22** (SSH)
* **OS disk:** Premium SSD **50 GB**

> Result: Jenkins available at `http://<azure-vm-public-ip>:8080` with Docker, Git, and Java ready for pipelines.

---

## 2) Prerequisites

* Azure subscription + permission to create resources
* SSH client on your workstation
* (Optional) Azure CLI installed and logged in: `az login`

---

## 3) Create the Azure VM

### Option A — Azure Portal

1. **Create resource group** (e.g., `rg-jenkins`).
2. **Create VM** → Ubuntu Server 22.04/24.04 LTS

   * **Name:** `jenkins`
   * **Size:** Standard_D2s_v3 (2 vCPU, 8 GB) or similar
   * **Authentication type:** SSH public key
   * **Key pair:** Upload/use the public key that matches **`jenkins.pem`** (private key kept locally)
   * **OS disk:** Premium SSD, **50 GB**
3. **Networking / NSG rules:**

   * Allow **SSH (22)** from your IP
   * Add inbound rule for **TCP 8080** (Source: your IP or required range)
4. **Review + Create**.

### Option B — Azure CLI (optional)

```bash
# Create RG
az group create -n rg-jenkins -l centralindia

# Create VM (Ubuntu LTS)
az vm create \
  -g rg-jenkins -n jenkins \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username atul \
  --ssh-key-values ~/.ssh/jenkins.pub \
  --os-disk-size-gb 50 \
  --public-ip-sku Standard

# Open port 8080 in the NSG
az vm open-port -g rg-jenkins -n jenkins --port 8080 --priority 1010
```

---

## 4) Connect to the VM

On your laptop/mac:

```bash
cd ~/Downloads
chmod 400 jenkins.pem
ssh -i jenkins.pem atul@<public-ip>
# Example from your notes:
# ssh -i jenkins.pem atul@40.90.232.193
```

(Optional) Set hostname:

```bash
sudo hostnamectl set-hostname jenkins && exec $SHELL -l
```

---

## 5) Base updates & essentials

```bash
sudo apt update -y
sudo apt install -y ca-certificates curl gnupg lsb-release software-properties-common
```

---

## 6) Install Java (JRE 21) & verify

```bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

> Jenkins LTS works with Java 21. Java 17 is also commonly used if you prefer an LTS baseline.

---

## 7) Install Git & Docker

```bash
sudo apt install -y git docker.io tree

# Verify versions
docker --version
git --version

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# (Optional) Docker Hub login if you pull/push private images
# sudo docker login

# Configure Git identity
git config --global user.name "Atul Kamble"
git config --global user.email "atul_kamble@hotmail.com"
git config --list
```

> On Ubuntu, `docker.io` package is fine for demos. For the latest Docker Engine, use Docker’s official repo.

---

## 8) Install Jenkins (Debian/Ubuntu repo)

1. Add the Jenkins repo key and entry:

```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

2. Install Jenkins:

```bash
sudo apt update
sudo apt install -y jenkins
```

3. Allow Jenkins to use Docker:

```bash
sudo usermod -aG docker jenkins
```

4. Start & enable Jenkins service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins --no-pager
```

> **Important:** After adding `jenkins` to the `docker` group, restart Jenkins so the group membership is picked up:

```bash
sudo systemctl restart jenkins
```

---

## 9) (Optional) UFW firewall rules on VM

If UFW is enabled on Ubuntu:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw enable    # only if not already enabled; confirm when prompted
sudo ufw status
```

The NSG must also allow 8080 as configured earlier.

---

## 10) Access Jenkins UI & initial setup

1. Open in your browser:

```
http://<azure-vm-public-ip>:8080
```

2. Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

3. Paste in the UI → Choose **Install suggested plugins** (recommended).
4. Create **admin user** (username + strong password) and confirm Jenkins URL.

---

## 11) Recommended plugins

* **Docker**
* **Docker Pipeline**
* **Blue Ocean** (optional, modern UI)
* (Optional) **GitHub**, **Pipeline: Stage View**, **Credentials Binding**

---

## 12) Global Tool Configuration

Navigate: **Manage Jenkins → Tools**. Add tool names and enable auto‑install where available:

* **JDK:** `MyJDK` (adoptopenjdk/temurin; auto‑install or point to `/usr/lib/jvm/java-21-openjdk-amd64`)
* **Maven:** `MyMaven` (auto‑install)
* **Gradle:** `MyGradle` (auto‑install)
* **Ant:** `MyAnt` (auto‑install)
* **Docker:** `MyDocker` (CLI typically at `/usr/bin/docker`)

> Ensure the Jenkins user can run `docker` without `sudo` (group step above). Test: **Manage Jenkins → Script Console** → `println "${"id".execute().text}"` or run a Freestyle job shell: `docker --version`.

---

## 13) Create your first Pipeline job

**New Item → Pipeline** → Name: `hello-pipeline` → Definition: **Pipeline script**.

**Pipeline script (declarative):**

```groovy
pipeline {
  agent any
  stages {
    stage('Development') {
      steps {
        echo 'I am in development stage'
        sh 'git --version'
      }
    }
    stage('Testing') {
      steps {
        echo 'I am in testing stage'
        sh 'docker --version'
      }
    }
    stage('Production') {
      steps {
        echo 'I am in production stage'
        sh 'java --version'
      }
    }
  }
}
```

Save → **Build Now**.

---

## 14) Validate & Console Output

Open the build → **Console Output**. You should see:

* Jenkins pipeline stage logs
* Versions printed for **Git**, **Docker**, **Java**

---

## 15) Common Troubleshooting

* **Port 8080 not reachable:**

  * Check NSG inbound rule for 8080
  * Check VM firewall (UFW) rules
  * `sudo systemctl status jenkins` to confirm service is running
* **`docker: permission denied` in jobs:**

  * Ensure `jenkins` is in `docker` group: `id jenkins`
  * Restart Jenkins: `sudo systemctl restart jenkins`
* **Java errors on startup:**

  * Confirm Java installed: `java -version`
  * If needed, switch to Java 17: `sudo apt install -y openjdk-17-jre`
* **Low disk space:**

  * Use at least **50 GB** OS disk; clean apt cache `sudo apt autoremove && sudo apt clean`
* **Slow plugin downloads:**

  * Verify outbound internet access; consider switching mirrors

---

## 16) Basic Security & Hardening (recommended)

* Change admin password; store credentials in Azure Key Vault
* Restrict NSG 8080 source IP to your office/home IPs
* Consider reverse proxy + HTTPS (Nginx) and disable direct 8080 exposure
* Regularly update Jenkins and plugins
* Enable backups of `/var/lib/jenkins` (jobs, config, secrets)

---

## 17) Clean Removal (optional)

```bash
sudo systemctl disable --now jenkins
sudo apt purge -y jenkins
sudo rm -rf /var/lib/jenkins /var/log/jenkins /etc/default/jenkins
```

---

## 18) Quick Reference (from your notes)

```bash
# Updates
sudo apt update -y

# Java 21
sudo apt install -y fontconfig openjdk-21-jre && java -version

# Git, Docker, Tree
sudo apt install -y git docker.io tree
sudo systemctl start docker && sudo systemctl enable docker

# Jenkins repo + install
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update && sudo apt install -y jenkins

# Docker group for Jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

**You’re all set!** Open `http://<public-ip>:8080`, finish the setup wizard, install suggested plugins, add Docker/Maven/Gradle tools, and run the sample pipeline.
