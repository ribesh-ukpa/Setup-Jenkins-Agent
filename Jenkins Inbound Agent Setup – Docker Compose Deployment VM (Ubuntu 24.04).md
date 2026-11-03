# Jenkins Inbound Agent Setup – Docker Compose Deployment VM (Ubuntu 24.04)

## 1. Purpose and end-state

This guide connects a new Ubuntu 24.04 VM, running Docker, to Jenkins as a permanent **inbound agent** over WebSocket, so pipelines can deploy containers on that VM with `docker compose`. Follow it top to bottom; every step has a check.

When finished:

1. The VM connects **out** to the Jenkins controller over HTTPS. Jenkins never connects in, so no SSH rule or inbound port is needed for Jenkins.
2. A systemd service (`jenkins-agent`) runs the agent as a dedicated `jenkins` user and restarts it after a reboot.
3. The node shows **online** in Jenkins with a label that only deployment jobs use.
4. A pipeline using that label runs `docker` and `docker compose` on the VM without permission errors.

This method is called **Launch agent by connecting it to the controller** in the Jenkins UI (formerly JNLP). It differs from the SSH method, where the controller opens the connection to the agent.

## 2. Prerequisites and values to decide first

Agree these values before starting, and use them consistently in every step. Never write the node secret in this document, a ticket or a chat.

| Placeholder | Meaning | Example / note |
| --- | --- | --- |
| `<JENKINS_URL>` | Public URL of the controller, with trailing slash | `https://jenkins.example.com/` (must match Manage Jenkins > System > Jenkins URL) |
| `<NODE_NAME>` | Node name in Jenkins | Use a new, descriptive name. It must match exactly, including case |
| `<NODE_LABEL>` | Label pipelines use to target this VM | Must be unique to this VM unless you deliberately share it (see section 7) |
| `<AGENT_DIR>` | Remote root directory on the VM | `/home/jenkins/agent` |

You also need:

- An Ubuntu 24.04 VM with Docker Engine and the Docker Compose plugin already installed (tested with Docker Engine 29.x and Compose v5.x).
- `sudo` access on the VM.
- Permission to create nodes in Jenkins (Manage Jenkins > Nodes).
- Access to the controller host (or the person who owns it) to review the reverse-proxy configuration in section 5.
- The **Java major version the controller runs**. The agent must use the same major version or newer. Check it with `java -version` on the controller, or look at the Jenkins version and Java version at the bottom of Manage Jenkins > System Information. In this setup the controller runs Java 21.

## 3. Network and security

The agent opens an outbound HTTPS (WebSocket) connection to the controller, so the VM only needs to reach `<JENKINS_URL>`. Test this first from the VM:

```bash
curl -I <JENKINS_URL>login
```

A `200` response means the route works. If it fails, fix DNS, outbound rules or the controller's firewall before going further.

Rules to follow:

- **No new inbound rule is needed** on the VM's network security group for Jenkins. Do not open ports for the agent. Azure allows outbound traffic by default. If the controller side restricts source IPs, ask the controller owner to allow this VM's public IP, through your normal NSG approval process.
- **Docker group membership is effectively root.** Anyone in the `docker` group can control the host. Only the `jenkins` user and named administrators should be in it. Record this in your Cyber Essentials access-control evidence if the VM is in scope.
- **Do not publish databases and queues to the internet.** If compose files bind services such as PostgreSQL, MongoDB, Redis or Kafka to `0.0.0.0`, change them to `127.0.0.1:<port>:<port>` unless another host genuinely needs them, and keep the NSG deny-by-default.
- **Remove SSH rules you no longer need.** If an old rule allowed the controller to SSH into this VM, delete it once the inbound agent works.
- **The node secret is a credential.** Treat it like a password (section 11).

## 4. Step 1: Prepare the VM

Run these commands on the new VM.

**1. Install Java.** The agent's Java must be the **same major version as the controller, or newer**. If the VM has an older Java, the agent connects and then fails with `UnsupportedClassVersionError ... class file version 65.0`, and no builds can run. Class file version 65 means the controller runs Java 21 (61 means Java 17).

First find the controller's Java version: run `java -version` on the controller host, or check `java.version` under Manage Jenkins > System Information. This setup uses Java 21, so the commands below install 21. If your controller runs a different version, change the package name to match, for example `openjdk-17-jre-headless`.

```bash
sudo apt update
sudo apt install -y openjdk-21-jre-headless git curl
sudo update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java
java -version
```

The last command must report version 21. If `--set` cannot find that path, run `sudo update-alternatives --config java` and choose the Java 21 entry.

**2. Create the dedicated agent user and give it Docker access.**

```bash
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins
sudo mkdir -p /home/jenkins/agent /opt/jenkins-agent /etc/jenkins-agent
sudo chown -R jenkins:jenkins /home/jenkins/agent /opt/jenkins-agent
```

If you also want your own administrator account to use Docker without `sudo`, add it with `sudo usermod -aG docker <your-user>` and log out and in again. Only do this for named administrators.

**3. Check Docker works for the agent user.**

```bash
sudo -u jenkins -H docker version
sudo -u jenkins -H docker compose version
sudo -u jenkins -H docker ps
```

All three must run without a `permission denied` error. If the Compose command is missing, install the plugin with `sudo apt install docker-compose-plugin` (or follow Docker's repository instructions for your install).

**4. Download the agent.** The controller serves the matching agent JAR itself, so the versions always agree.

```bash
sudo -u jenkins curl -fsSL <JENKINS_URL>jnlpJars/agent.jar -o /opt/jenkins-agent/agent.jar
```

If you run this as another user, make sure `/opt/jenkins-agent/agent.jar` ends up readable by `jenkins`.

## 5. Step 2: Allow WebSocket through the reverse proxy (controller host)

If Jenkins sits behind nginx, the proxy must pass the WebSocket upgrade headers. Without them the agent fails with `Failed to connect: Handshake error` and the controller logs `incomplete headers` for `jenkins.agents.WebSocketAgents#doIndex`, with no `Upgrade` header in the list. This was the first blocker in the original setup.

Do this once per controller. It is not needed again for further agents. **A change to the live proxy needs review by the controller's owner before it is applied.**

1. Find the Jenkins server block:

```bash
sudo grep -rln "<jenkins-hostname>" /etc/nginx/
sudo nginx -T 2>/dev/null | grep -n -i -E "server_name|proxy_pass|upgrade|connection"
```

2. Make sure a connection-upgrade map exists once at `http` level (many setups already have it). If it does not, add:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

3. Inside the Jenkins `location` block, next to `proxy_pass` and `proxy_http_version 1.1;`, add:

```nginx
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

4. Back up, test and reload (a reload is graceful and does not drop sessions):

```bash
sudo cp /etc/nginx/<file> /etc/nginx/<file>.bak-$(date +%F)
sudo nginx -t && sudo systemctl reload nginx
```

Also check **Manage Jenkins > System > Jenkins URL** is exactly the public URL (`<JENKINS_URL>`), because the agent uses it.

Do not test the WebSocket endpoint with plain `curl`. It is not a WebSocket client, and the endpoint answers `400 This endpoint is only for use from agent.jar in WebSocket mode`, which proves nothing. Use the real agent (section 7) and the controller log (section 10).

## 6. Step 3: Create the node in Jenkins

In Jenkins go to **Manage Jenkins > Nodes > New Node**, choose **Permanent Agent**, and set:

| Field | Value |
| --- | --- |
| Node name | `<NODE_NAME>` |
| Number of executors | 1 or 2 (one deployment at a time is safest for compose deploys) |
| Remote root directory | `/home/jenkins/agent` |
| Labels | `<NODE_LABEL>` |
| Usage | Only build jobs with label expressions matching this node |
| Launch method | **Launch agent by connecting it to the controller** |
| Use WebSocket | Tick it if the option is shown (under Advanced on some versions) |
| Availability | Keep this agent online as much as possible |

Save, then open the node's page. It shows an agent command and a **secret**. Check the command contains `-webSocket`. If it does not, the node is not configured for WebSocket, so look for the checkbox under the Launch method setting.

Copy the secret straight into the secure step in section 7. Do not paste it into chat, tickets or this document.

**Secrets cannot be regenerated.** Jenkins derives a node's secret from its name. If a secret leaks, the practical fix is to create a new node with a new name and delete the old one (section 11).

## 7. Step 4: Run the agent as a systemd service

**1. Store the secret in a root-owned file.** The prompt below hides what you paste and keeps the secret out of your shell history and out of `ps` output.

```bash
read -rs -p 'Node secret: ' S; echo
printf %s "$S" | sudo tee /etc/jenkins-agent/secret >/dev/null
unset S
sudo chown root:jenkins /etc/jenkins-agent/secret
sudo chmod 640 /etc/jenkins-agent/secret
sudo wc -c /etc/jenkins-agent/secret
```

`wc -c` must print 64. Another number means the secret was pasted incorrectly, so repeat the step.

**2. Create the service file** with `sudo vim /etc/systemd/system/jenkins-agent.service`. Do not use plain `touch` or `>` without `sudo`, because `/etc/systemd/system` is root-only.

```ini
[Unit]
Description=Jenkins inbound agent (<NODE_NAME>)
After=network-online.target docker.service
Wants=network-online.target

[Service]
User=jenkins
Group=jenkins
WorkingDirectory=/home/jenkins/agent
ExecStart=/usr/bin/java -jar /opt/jenkins-agent/agent.jar \
  -url <JENKINS_URL> \
  -name <NODE_NAME> \
  -secret @/etc/jenkins-agent/secret \
  -webSocket \
  -workDir /home/jenkins/agent
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

The `@/etc/jenkins-agent/secret` form makes the agent read the secret from the file, so it never appears on the command line.

**3. Start it.**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins-agent
sudo systemctl status jenkins-agent --no-pager
sudo journalctl -u jenkins-agent -n 20 --no-pager
```

A healthy start logs `WebSocket connection open` and `Connected`. Do not enable debug logging on this service: debug output prints request headers, including the secret.

## 8. Step 5: Verify the connection and run a smoke test

**1. Check the node is online.** In Manage Jenkins > Nodes the node must show as connected, with architecture, free disk space and response time filled in. Open the node page and read the log if it does not.

**2. Run a throwaway pipeline** that targets the node. It proves the agent user, Java and Docker all work end to end:

```groovy
pipeline {
  agent { label '<NODE_LABEL>' }
  stages {
    stage('Check') {
      steps {
        sh 'whoami && java -version && docker version --format "{{.Server.Version}}" && docker compose version'
      }
    }
  }
}
```

The build log must say `Running on <NODE_NAME>` (not `Running on Jenkins`, which is the built-in controller node). `whoami` must print `jenkins`, and the Docker commands must succeed without permission errors.

**3. Check it survives a restart.**

```bash
sudo systemctl restart jenkins-agent
sudo journalctl -u jenkins-agent -n 10 --no-pager
```

The node must come back online within about a minute. A full VM reboot should do the same, because the service is enabled.

## 9. Step 6: Point pipelines at the node

A job only runs on this VM if its pipeline asks for the node's label. Jobs with `agent any` or no label run on the controller, which is the wrong place for deployments.

```groovy
agent { label '<NODE_LABEL>' }
```

If the pipeline's `agent` is set inside a shared library function, change it there, and review the change first because every pipeline that loads that library is affected.

**Keep the label unique to this VM.** If another node carries the same label, Jenkins can send a deployment to either node without any error, and a deploy can land on the wrong host. To check, open the node page and click the label name: Jenkins lists every node that has it. Use a distinct label such as `<base-label>-<vm-name>` unless you intend to replace the old node or share the load.

Deployment notes for `docker compose` on this VM:

- Keep the compose project in a fixed directory owned by `jenkins`, for example `/opt/apps/<app>`, rather than inside the workspace, so containers and volumes survive workspace cleanup.
- Log in to the registry inside the pipeline with a Jenkins credential (`withCredentials` plus `docker login`) instead of storing a token on the VM.
- Prefer deploying a specific image tag or digest over `latest`, so a redeploy is repeatable and a rollback is a known value.
- Check that the Jenkinsfile and shared library work on this host: paths, compose files, environment files and secrets copied from an older repo or host may not exist here.

## 10. Troubleshooting

Start with the agent log on the VM (`sudo journalctl -u jenkins-agent -n 30 --no-pager`) and the controller log (Manage Jenkins > System Log, or `sudo journalctl -u jenkins --since "30 minutes ago"`). The controller log usually says why a connection was refused.

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Agent logs `Failed to connect: Handshake error` repeatedly; controller logs `incomplete headers` without `Upgrade` | Reverse proxy drops the WebSocket upgrade headers | Add `Upgrade` and `Connection` headers in the Jenkins nginx block (section 5) |
| Agent connects, then `UnsupportedClassVersionError ... class file version 65.0` | Agent Java is older than the controller's | Install the controller's Java major version on the VM (section 4) |
| `permission denied ... /var/run/docker.sock` | User is not in the `docker` group, or has not logged in again since being added | `sudo usermod -aG docker <user>`; log out and in. For the service, restart it |
| `AccessDeniedException: /home/jenkins/agent` | Agent started manually as a different user | Run as `jenkins` (`sudo -u jenkins -H ...`) or use the service |
| `Permission denied` creating the service file | `/etc/systemd/system` is root-only | Use `sudo vim` or `sudo tee`, then `sudo systemctl daemon-reload` |
| Handshake error with a correct proxy | Wrong secret, or node name differs in case or spelling | Recheck the 64-character secret file and the `-name` value in the service |
| Build log says `Running on Jenkins` | Pipeline does not request the node's label | Set `agent { label '<NODE_LABEL>' }` (section 9) |
| Node drops after about a minute | Proxy read timeout too short | Raise `proxy_read_timeout` on the Jenkins block and reload nginx |

Testing the WebSocket endpoint with plain `curl` always returns `400 This endpoint is only for use from agent.jar in WebSocket mode`, even when everything is correct. It is not a useful test.

## 11. Secret handling and maintenance

**If the secret is exposed** (pasted in chat, shown in debug output or a ticket), treat it as compromised. Jenkins derives a node's secret from its name, so the fix is to replace the node:

1. In Jenkins, create a new node (the **Copy existing Node** option keeps the settings) with a new name, and copy its new secret.
2. Delete the old node.
3. On the VM, update the secret file and the `-name` value, then restart:

```bash
read -rs -p 'New secret: ' S; echo
printf %s "$S" | sudo tee /etc/jenkins-agent/secret >/dev/null; unset S
sudo chown root:jenkins /etc/jenkins-agent/secret && sudo chmod 640 /etc/jenkins-agent/secret
sudo sed -i 's/-name <OLD_NAME>/-name <NEW_NAME>/' /etc/systemd/system/jenkins-agent.service
sudo systemctl daemon-reload && sudo systemctl restart jenkins-agent
sudo journalctl -u jenkins-agent -n 15 --no-pager
```

Also update `Description=` in the service file if you want it to show the new name.

**Logging.** Never enable debug logging (`-loggingConfig` with level `ALL`) on the running service, and never paste raw agent debug output anywhere. It contains the secret. When sharing logs for help, filter first and remove secrets, internal IPs and customer identifiers.

**Routine maintenance.**

- Keep Java and Docker patched through your normal patching process, and re-run the smoke test after major upgrades.
- Jenkins and the agent JAR should stay on matching versions. After a controller upgrade, download the agent JAR again (section 4, step 4) and restart the service.
- Watch disk space. Old images and build workspaces grow over time, so schedule `docker system prune` or similar housekeeping that suits your deployments.
- When decommissioning the VM, delete the node in Jenkins, stop and disable the service, and remove `/etc/jenkins-agent/secret`.

## 12. Review checklist and open items

This is internal work product. It should be reviewed by a senior engineer or the controller's owner before other people rely on it, and any change to the production controller or reverse proxy needs their approval.

**Completion checklist** (tick when verified):

- [ ] VM reaches `<JENKINS_URL>` over HTTPS
- [ ] Java major version matches the controller
- [ ] `jenkins` user can run `docker` and `docker compose`
- [ ] Reverse proxy passes `Upgrade` and `Connection` headers (reviewed, backed up, reloaded)
- [ ] Node created with the agreed name, label and WebSocket enabled
- [ ] Secret stored in a root-owned file with mode 640 and not shared anywhere
- [ ] `jenkins-agent` service enabled and node shows online
- [ ] Smoke-test pipeline ran on `<NODE_NAME>` as the `jenkins` user
- [ ] Label is unique to this VM, or sharing it is a recorded decision
- [ ] Unused SSH rules for this VM removed from the NSG
- [ ] Databases and queues bound to `127.0.0.1` unless remote access is required
