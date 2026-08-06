# Jenkins Agent Node Setup via SSH (Key-Based Authentication)

## Overview

This guide covers setting up Jenkins agent nodes on remote VMs using SSH key-based authentication only — no password access required. This version supports **non-standard SSH ports** (e.g., port 2222) commonly used in production environments.

```
Jenkins Controller                    Agent VM
──────────────────                    ────────
Private Key (in Jenkins Credentials)  Public Key (in ~/.ssh/authorized_keys)
     │                                      │
     └──────── SSH connection ──────────────┘
               (no password needed)
               (supports port 2222)
```

---

## Prerequisites

- Jenkins controller is running and accessible
- Agent VM is reachable over the network on the specified SSH port
- Both VMs have the **same Java version** installed (e.g., Java 21)
- SSH is enabled on the agent VM and listening on your custom port (e.g., 2222)

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

### Configure SSH to Listen on Custom Port (e.g., 2222)

If your SSH is running on a non-standard port, verify it on the agent VM:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure this line is present:

```
Port 2222
```

Then restart SSH:

```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

Verify it's listening:

```bash
sudo netstat -tulpn | grep 2222
# or
sudo ss -tulpn | grep 2222
```

---

## (Optional-If already setup once don't touch : GO TO Step 3) Step 2: Generate SSH Key Pair on the Controller

Run the following **on the Jenkins controller** as the jenkins user:

```bash
sudo -u jenkins bash

mkdir -p /var/lib/jenkins/.ssh
ssh-keygen -t ed25519 -C "jenkins-agent" -f /var/lib/jenkins/.ssh/agent_key -N ""
```

**Important:** The `-N ""` flag creates the key **without a passphrase**. This is essential because Jenkins running as a service cannot provide passphrases interactively.

⚠️ **If you already have a passphrase-protected key**, remove it:

```bash
sudo -u jenkins ssh-keygen -p -N "" -m pem -f /var/lib/jenkins/.ssh/agent_key
# Enter the current passphrase when prompted
```

This creates:

- `/var/lib/jenkins/.ssh/agent_key` — **private key** (stays on controller, no passphrase)
- `/var/lib/jenkins/.ssh/agent_key.pub` — **public key** (goes to agent VM)

---

## Step 3: Copy Public Key to the Agent VM

Get the public key content from the **controller**:

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

From the **controller**, Add the connection to your custom port of the AGENT:
```bash
ssh-keyscan -p 2222 <AGENT_VM_IP> | sudo tee -a /var/lib/jenkins/.ssh/known_hosts > /dev/null
chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

```bash
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/agent_key jenkins@<AGENT_VM_IP> -p 2222

# It will ask for PASSPHASE if it works
```

**Expected result:** You should land directly in the agent shell with **no password prompt**. If this works, Jenkins will connect successfully too.

If it fails, check:
- Agent SSH is listening on port 2222: `ssh -v` shows the port connection
- Public key is in `~/.ssh/authorized_keys` on the agent
- File permissions are correct (see Quick Reference below)

If it doesn't work try this:
```bash
# Run this as a normal user not jenkins

```

---

## Step 5: Add Private Key to Jenkins Credentials

1. Go to **Manage Jenkins → Credentials → System → Global → Add Credentials**
2. Fill in the following:

| Field       | Value                                                                |
| ----------- | -------------------------------------------------------------------- |
| Kind        | SSH Username with private key                                        |
| ID          | `jenkins-agent-ssh-key`                                              |
| Username    | `jenkins`                                                            |
| Private Key | Enter directly → paste contents of `/var/lib/jenkins/.ssh/agent_key` |
| Passphrase  | Leave empty (key should have no passphrase)                          |

Get the private key with:

```bash
sudo cat /var/lib/jenkins/.ssh/agent_key
```

Copy the entire output including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`.

---

## Step 6: Install the SSH Build Agents Plugin

1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search for **SSH Build Agents**
3. Install and restart Jenkins

---

## Step 7: Add the Agent Node in Jenkins

1. Go to **Manage Jenkins → Nodes and Clouds → New Node**
2. Enter a node name (e.g., `agent-1`) and select **Permanent Agent**
3. Configure the node:

| Field                 | Value                          |
| --------------------- | ------------------------------ |
| # of executors        | 2 (or match CPU cores)         |
| Remote root directory | `/home/jenkins/agents`         |
| Labels                | e.g., `linux ubuntu`           |
| Launch method         | Launch agents via SSH          |
| Host                  | `<AGENT_VM_IP>`                |
| Port                  | `2222` (your custom SSH port)  |
| Credentials           | Select the key added in Step 5 |
| Host Key Verification | See options below              |

### Host Key Verification Strategy

Click **Advanced** to access the Host Key Verification Strategy option.

| Strategy                              | When to Use                                           |
| ------------------------------------- | ----------------------------------------------------- |
| **Non verifying**                     | Quick setup / initial testing                         |
| **Manually Trusted Key Verification** | First-time setup (recommends trusting the host)      |
| **Known hosts file**                  | Production (more secure, requires setup below)       |

#### (SKIP ALREADY DONE ABOVE) For Production: Use Known Hosts File Strategy

To use the **Known hosts file** strategy, first populate it on the controller with the agent's host key **including the port**:

```bash
sudo -u jenkins bash
mkdir -p /var/lib/jenkins/.ssh
```

```bash
# Add host key for custom port (without -H flag to avoid hashing issues)
# Run this as a normal user not jenkins
ssh-keyscan -p 2222 <AGENT_VM_IP> | sudo tee -a /var/lib/jenkins/.ssh/known_hosts > /dev/null
chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

Verify the entry was added:

```bash
sudo -u jenkins tail -5 /var/lib/jenkins/.ssh/known_hosts
```

You should see entries like:

```
[<AGENT_VM_IP>]:2222 ssh-rsa AAAAB3NzaC...
[<AGENT_VM_IP>]:2222 ecdsa-sha2-nistp256 AAAAE2Vjd...
[<AGENT_VM_IP>]:2222 ssh-ed25519 AAAAC3NzaC...
```

4. Click **Save**, then **Launch Agent**
5. Check the **Log** tab — you should see:
   ```
   [SSH] Opening SSH connection to <AGENT_VM_IP>:2222.
   [SSH] Connected to <AGENT_VM_IP>:2222
   Agent successfully connected and online
   ```

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

Then restart SSH:

```bash
sudo systemctl restart sshd
```

> ⚠️ **Important:** Confirm that key-based SSH login works (Step 4) **before** disabling password auth, or you may lock yourself out.

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
| --- | --- | --- |
| `Connection refused` | SSH not running on specified port | Verify SSH is listening on port 2222: `sudo ss -tulpn \| grep 2222` |
| `Permission denied (publickey)` | Key has passphrase or public key not in authorized_keys | Remove passphrase: `ssh-keygen -p -N "" -f /var/lib/jenkins/.ssh/agent_key` |
| `Enter passphrase for key` | Private key has a passphrase (won't work with Jenkins) | Remove passphrase using command above |
| `No entry currently exists in the Known Hosts file` | Host key not added for custom port | Run: `ssh-keyscan -p 2222 <IP> \| sudo tee -a /var/lib/jenkins/.ssh/known_hosts > /dev/null` |
| `UnsupportedClassVersionError` | Java version mismatch | Install same Java version on both controller and agent |
| Agent keeps disconnecting | Firewall blocking port 2222 | `sudo ufw allow 2222` and verify from controller: `nc -zv <AGENT_IP> 2222` |
| `Host key not found in known_hosts` | File doesn't exist or has wrong permissions | `sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh && sudo -u jenkins ssh-keyscan -p 2222 <IP> >> /var/lib/jenkins/.ssh/known_hosts` |

---

## Quick Reference: Key Permissions

Correct permission structure is critical for SSH to work:

```bash
# On agent VM — correct permission structure
chmod 700 /home/jenkins/.ssh
chmod 600 /home/jenkins/.ssh/authorized_keys
chown -R jenkins:jenkins /home/jenkins/.ssh

# On controller — correct permission structure
chmod 700 /var/lib/jenkins/.ssh
chmod 600 /var/lib/jenkins/.ssh/agent_key
chmod 644 /var/lib/jenkins/.ssh/agent_key.pub
chmod 644 /var/lib/jenkins/.ssh/known_hosts
chown -R jenkins:jenkins /var/lib/jenkins/.ssh
```

---

## Important Notes for Port 2222

✅ **Do:**
- Update the **Port** field in Jenkins node configuration to `2222`
- Use `ssh-keyscan -p 2222 <IP>` when adding to known_hosts
- Test with `ssh -p 2222` before configuring Jenkins
- Remove passphrases from SSH keys used for automation

❌ **Don't:**
- Forget the `-p 2222` flag in ssh commands
- Leave passphrases on keys used for Jenkins automation
- Use `-H` flag with ssh-keyscan if you need plain IP entries (use `-H` only for hashing)
- Disable password auth before testing key-based SSH works

---

## Testing Checklist

Before considering the setup complete:

- [ ] Agent VM SSH is listening on port 2222
- [ ] `sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/agent_key jenkins@<IP> -p 2222` works without passphrase
- [ ] `/var/lib/jenkins/.ssh/known_hosts` contains entry for `[<IP>]:2222`
- [ ] Jenkins credentials have no passphrase set
- [ ] Jenkins node configuration shows **Port: 2222**
- [ ] Agent is shown as **online** in Jenkins UI
- [ ] Test job runs successfully on the agent