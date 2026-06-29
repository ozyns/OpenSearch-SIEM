# OpenSearch-SIEM

A fully functional SIEM (Security Information and Event Management) solution built with OpenSearch, Logstash, and Winlogbeat.

## Overview

This project deploys a complete log collection and threat detection pipeline:

- **OpenSearch** - stores and indexes security logs
- **OpenSearch Dashboards** - visualizes logs and security events
- **Logstash** - receives and processes logs from agents
- **Winlogbeat** - collects Windows Event Logs from monitored machines
- **Sigma rules** - vendor-neutral detection rules converted to Lucene queries
- **OpenSearch Alerting** - real-time alerts sent to Slack on attack detection

> **Note:** Winlogbeat requires a Windows machine or VM.
> The Docker stack runs on any OS. For log collection on Linux machines,
> use Filebeat instead of Winlogbeat.

## Requirements

- **Windows**: Docker Desktop with WSL2 backend
- **Linux**: Docker Engine + Docker Compose plugin
- **macOS**: Docker Desktop
- Minimum 6GB RAM available for the stack
- A Windows machine or VM for Winlogbeat log collection

---

## Installation

### Windows

**1. Install Docker Desktop**

Download from https://www.docker.com/products/docker-desktop/

During installation:
- Enable **Use WSL 2 instead of Hyper-V**
- Leave **Allow Windows Containers** unchecked

Verify:
```powershell
docker --version
docker compose version
```

---

### Linux (Fedora)

**1. Install Docker Engine**

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start and enable Docker:
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Add yourself to the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:
```bash
docker --version
docker compose version
```

**2. Fix OpenSearch memory requirement (Linux only)**

OpenSearch requires a higher virtual memory limit on Linux or it will crash on startup:

```bash
sudo sysctl -w vm.max_map_count=262144
```

To make it permanent across reboots:
```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

Then apply the same `vm.max_map_count` fix above.

---

## Quick Start

**1. Clone the repository**

```bash
git clone https://github.com/ozyns/opensearch-siem.git
cd opensearch-siem
```

**2. Set the admin password**

In `docker-compose.yml`, replace `CHANGE_ME` with a strong password:

```yaml
- OPENSEARCH_INITIAL_ADMIN_PASSWORD=YourStrongPassword@123
```

Password must contain uppercase, lowercase, digit, and special character.

**3. Start the stack**

```bash
sudo chmod 666 /var/run/docker.sock
docker compose up -d
```

First run will download ~2-3GB of Docker images. Wait 1-2 minutes for all services to start.

**4. Verify the stack is running**

```bash
docker compose ps
```

All three containers should show `Up`:
- `opensearch-stack-opensearch-1`
- `opensearch-stack-opensearch-dashboards-1`
- `opensearch-stack-logstash-1`

**5. Test OpenSearch is responding**

```bash
curl http://localhost:9200
```

**6. Access OpenSearch Dashboards**

Open your browser:
http://localhost:5601

---

## Winlogbeat Setup (Windows Machine)

**1. Download Winlogbeat 7.17.0**

https://www.elastic.co/downloads/past-releases/winlogbeat-7-17-0

Download the **Windows ZIP x86_64** version.

**2. Extract to:**
C:\Program Files\Winlogbeat

**3. Edit `winlogbeat.yml`**

Comment out the Elasticsearch output:
```yaml
#output.elasticsearch:
  #hosts: ["localhost:9200"]
```

Enable the Logstash output:
```yaml
output.logstash:
  hosts: ["YOUR_HOST_IP:5044"]
```

Replace `YOUR_HOST_IP` with the IP of the machine running Docker.

**4. Install and start the service**

Open PowerShell as Administrator:
```powershell
cd "C:\Program Files\Winlogbeat"
Set-ExecutionPolicy Bypass -Scope Process -Force
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
Get-Service winlogbeat
```

Status should show `Running`.

**5. Allow port 5044 on the host firewall (Windows host only)**

```powershell
New-NetFirewallRule -DisplayName "Logstash Beats" -Direction Inbound -Protocol TCP -LocalPort 5044 -Action Allow
```

---

## Verify Log Ingestion

Check that indices are being created:

```bash
curl http://localhost:9200/_cat/indices?v
```

You should see `winlogbeat-*` indices appearing.

In OpenSearch Dashboards:
1. Go to **Stack Management → Index Patterns**
2. Create index pattern: `winlogbeat-*`
3. Time field: `@timestamp`
4. Go to **Discover** — Windows Event Logs will be visible

---

## Import the SOC Dashboard

1. Go to `http://localhost:5601`
2. Hamburger menu → **Stack Management → Saved Objects**
3. Click **Import**
4. Select `dashboard/soc-dashboard.ndjson`
5. Go to **Dashboard** → open **SOC Dashboard**

The dashboard includes:
- Failed Logins counter (EventID 4625)
- New Users Created counter (EventID 4720)
- Privilege Escalation counter (EventID 4672)
- PowerShell Executions counter (EventID 4104)
- Events Over Time by Event ID (bar chart)
- Failed Logins Over Time (bar chart)
- Top Event IDs distribution (pie chart)
- Top Source Hostnames (table)
- Top Usernames Targeted (table)

---

## Sigma Detection Rules

The `sigma-rules/` folder contains three detection rules:

| Rule | Event ID | MITRE Technique |
|---|---|---|
| `failed_login.yml` | 4625 | T1110 — Brute Force |
| `user_created.yml` | 4720 | T1136 — Create Account |
| `powershell_execution.yml` | 4104 | T1059.001 — PowerShell |

### Convert rules to OpenSearch Lucene

Install sigma-cli:
```bash
pip install sigma-cli
pip install pysigma-backend-opensearch
pip install pysigma-pipeline-windows
```

Convert a rule:
```bash
sigma convert -t opensearch_lucene -p windows-audit sigma-rules/failed_login.yml
```

Use the output query directly in OpenSearch Dashboards → Discover (with DQL off).

---

## Setting Up Alerts (Slack)

1. Create a Slack webhook at https://api.slack.com/apps
2. In OpenSearch Dashboards → **Alerting → Notifications → Channels**
3. Create a Slack channel with your webhook URL
4. Go to **Alerting → Monitors → Create monitor**
5. Configure a query-level monitor:
   - Index: `winlogbeat-*`
   - Filter: `winlog.event_id` = `4625`
   - Time range: 1 minute
   - Threshold: count > 50
   - Notification: your Slack channel

---

## Generate Test Events

To test the detection pipeline, run this in PowerShell on the monitored Windows machine:

```powershell
# Simulate failed logins (EventID 4625)
for($i=0; $i -lt 10; $i++) {
    $cred = New-Object System.Management.Automation.PSCredential("fakeuser", (ConvertTo-SecureString "wrongpass" -AsPlainText -Force))
    Start-Process cmd -Credential $cred -ErrorAction SilentlyContinue
}

# Simulate user creation (EventID 4720)
net user testuser Password123! /add
net user testuser /delete

# Simulate PowerShell execution (EventID 4104)
Set-ExecutionPolicy Bypass -Scope Process -Force
Invoke-Expression "Write-Output 'test'"
```

---

## Stack Versions

| Component | Version |
|---|---|
| OpenSearch | 2.13.0 |
| OpenSearch Dashboards | 2.13.0 |
| Logstash | 8.9.0 (OSS with OpenSearch plugin) |
| Winlogbeat | 7.17.0 |
