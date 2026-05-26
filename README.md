# Jenkins Agent Node Setup via SSH (Key-Based Authentication)

## Overview

This guide covers setting up Jenkins agent nodes on remote VMs using SSH key-based authentication only — no password access required.

```
Jenkins Controller                    Agent VM
──────────────────                    ────────
Private Key (in Jenkins Credentials)  Public Key (in ~/.ssh/authorized_keys)
     │                                      │
     └──────── SSH connection ──────────────┘
               (no password needed)
```

---

## Prerequisites

- Jenkins controller is running and accessible
- Agent VM is reachable over the network
- Both VMs have the **same Java version** installed (e.g., Java 21)
- SSH is enabled on the agent VM

> **Java Version Warning:** The controller and agent must run the same Java major version. A mismatch causes `UnsupportedClassVersionError` and the agent will fail to connect.

---

## Step 1: Prepare the Agent VM

SSH into the agent VM and run:

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install Java 21 (must match controller)
sudo apt install -y openjdk-21-jdk

# Verify
java -version
```

Create a dedicated Jenkins user and working directory:

```bash
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/agents
sudo chown jenkins:jenkins /home/jenkins/agents
```

---

## Step 2: Generate SSH Key Pair on the Controller

Run the following **on the Jenkins controller** as the jenkins user:

```bash
sudo -u jenkins bash

ssh-keygen -t ed25519 -C "jenkins-agent" -f /var/lib/jenkins/.ssh/agent_key
# Press Enter twice to skip passphrase
```

This creates:
- `/var/lib/jenkins/.ssh/agent_key` — **private key** (stays on controller)
- `/var/lib/jenkins/.ssh/agent_key.pub` — **public key** (goes to agent VM)

---

## Step 3: Copy Public Key to the Agent VM

Get the public key content from the controller:

```bash
sudo cat /var/lib/jenkins/.ssh/agent_key.pub
```

Then on the **agent VM**, add it to the jenkins user's authorized keys:

```bash
sudo -u jenkins bash

mkdir -p /home/jenkins/.ssh
chmod 700 /home/jenkins/.ssh

# Paste your public key below
echo "ssh-ed25519 AAAA...your-key..." >> /home/jenkins/.ssh/authorized_keys

chmod 600 /home/jenkins/.ssh/authorized_keys
chown -R jenkins:jenkins /home/jenkins/.ssh
```

---

## Step 4: Verify SSH Works Without a Password

From the **controller**, test the connection:

```bash
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/agent_key jenkins@<AGENT_VM_IP>
```

You should land directly in the agent shell with no password prompt. If this works, Jenkins will connect successfully too.

---

## Step 5: Add Private Key to Jenkins Credentials

1. Go to **Manage Jenkins → Credentials → System → Global → Add Credentials**
2. Fill in the following:

| Field | Value |
|---|---|
| Kind | SSH Username with private key |
| ID | `jenkins-agent-ssh-key` |
| Username | `jenkins` |
| Private Key | Enter directly → paste contents of `/var/lib/jenkins/.ssh/agent_key` |

Get the private key with:

```bash
sudo cat /var/lib/jenkins/.ssh/agent_key
```

---

## Step 6: Install the SSH Build Agents Plugin

1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search for **SSH Build Agents**
3. Install and restart Jenkins

---

## Step 7: Add the Agent Node in Jenkins

1. Go to **Manage Jenkins → Nodes → New Node**
2. Enter a node name (e.g., `agent-1`) and select **Permanent Agent**
3. Configure the node:

| Field | Value |
|---|---|
| # of executors | 2 (or match CPU cores) |
| Remote root directory | `/home/jenkins/agents` |
| Labels | e.g., `linux ubuntu` |
| Launch method | Launch agents via SSH |
| Host | `<AGENT_VM_IP>` |
| Credentials | Select the key added in Step 5 |
| Host Key Verification | See options below |

### Host Key Verification Options

| Strategy | When to Use |
|---|---|
| **Non verifying** | Quick setup / dev environments |
| **Known hosts file** | Production (more secure) |

To use **Known hosts file** strategy, first populate it on the controller:

```bash
sudo -u jenkins bash
mkdir -p /var/lib/jenkins/.ssh
ssh-keyscan -H <AGENT_VM_IP> >> /var/lib/jenkins/.ssh/known_hosts
chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

4. Click **Save**, then **Launch Agent**
5. Check the **Log** tab — you should see `Agent successfully connected and online`

---

## Step 8: Disable Password Authentication on Agent VM (Recommended)

Since only key-based auth is required, lock down SSH on the agent VM:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:

```
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

```bash
sudo systemctl restart sshd
```

> ⚠️ Confirm that key-based SSH login works (Step 4) **before** disabling password auth, or you may lock yourself out.

---

## Running Pipelines on the Agent

Target the agent in your Jenkinsfile using its label:

```groovy
pipeline {
    agent { label 'linux' }
    stages {
        stage('Build') {
            steps {
                sh 'echo Running on agent VM'
            }
        }
    }
}
```

### Running on Multiple Agents in Parallel

```groovy
pipeline {
    agent none
    stages {
        stage('Deploy to all agents') {
            parallel {
                stage('agent-0') {
                    agent { label 'agent-0' }
                    steps {
                        sh 'docker run -d --name nginx nginx:latest'
                    }
                }
                stage('agent-1') {
                    agent { label 'agent-1' }
                    steps {
                        sh 'docker run -d --name nginx nginx:latest'
                    }
                }
            }
        }
    }
}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `No Known Hosts file found` | known_hosts missing | Run `ssh-keyscan` (see Step 7) or switch to Non verifying strategy |
| `UnsupportedClassVersionError` | Java version mismatch | Install same Java version on both controller and agent |
| `Connection refused` | SSH not running on agent | `sudo systemctl start ssh` |
| `Permission denied` | Wrong key permissions | `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` |
| Agent keeps disconnecting | Firewall blocking | `sudo ufw allow 22` |

---

## Quick Reference: Key Permissions

```bash
# On agent VM — correct permission structure
chmod 700 /home/jenkins/.ssh
chmod 600 /home/jenkins/.ssh/authorized_keys
chown -R jenkins:jenkins /home/jenkins/.ssh

# On controller — correct permission structure
chmod 700 /var/lib/jenkins/.ssh
chmod 600 /var/lib/jenkins/.ssh/agent_key
chmod 644 /var/lib/jenkins/.ssh/agent_key.pub
chown -R jenkins:jenkins /var/lib/jenkins/.ssh
```
