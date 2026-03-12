# 🛡️ Wazuh SIEM — Complete Setup Guide

> A full walkthrough of deploying **Wazuh 4.14** on Ubuntu 22.04, adding Linux & Windows agents, enabling File Integrity Monitoring, configuring ModSecurity WAF, Suricata IDS, and more — documented step-by-step so anyone can replicate it.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [1. Server Setup (All Nodes)](#1-server-setup-all-nodes)
- [2. Set Static IPs](#2-set-static-ips)
- [3. Open Required Firewall Ports](#3-open-required-firewall-ports)
- [4. Install Wazuh Indexer](#4-install-wazuh-indexer)
- [5. Install Wazuh Server (Manager)](#5-install-wazuh-server-manager)
- [6. Install Wazuh Dashboard](#6-install-wazuh-dashboard)
- [7. Add Agents](#7-add-agents)
  - [Ubuntu/Linux Agent](#ubuntulinux-agent)
  - [Windows Agent](#windows-agent)
- [8. File Integrity Monitoring (FIM)](#8-file-integrity-monitoring-fim)
- [9. Generate Test Alerts](#9-generate-test-alerts)
- [10. Apache + ModSecurity WAF](#10-apache--modsecurity-waf)
- [11. Suricata IDS Integration](#11-suricata-ids-integration)
- [12. Enable Archive Logs](#12-enable-archive-logs)
- [13. Backup & Upgrade](#13-backup--upgrade)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│           Wazuh Server (All-in-One)      │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
│  │ Indexer  │ │ Manager  │ │Dashboard│  │
│  │  :9200   │ │  :55000  │ │  :443   │  │
│  └──────────┘ └──────────┘ └─────────┘  │
│         IP: 192.168.145.160              │
└─────────────┬───────────────────────────┘
              │  Wazuh Agent (port 1514/1515)
   ┌──────────┴────────────┐
   │                       │
┌──▼────────────┐   ┌──────▼──────────┐
│ Ubuntu Agent  │   │ Windows Agent   │
│ 192.168.145.X │   │ 192.168.145.X   │
└───────────────┘   └─────────────────┘
```

---

## Prerequisites

- Stable internet connection
- VMware (or any hypervisor)
- **3 VMs** running Ubuntu 22.04 Server:
  - 1× Wazuh Server (Indexer + Manager + Dashboard)
  - 1× Ubuntu Agent
  - 1× Windows Agent (optional but recommended)

---

## 1. Server Setup (All Nodes)

Run these on **every server** before anything else:

```bash
# Install essential tools
sudo apt update
sudo apt install net-tools -y

# Install SSH if not already present
sudo apt install openssh-server -y
```

---

## 2. Set Static IPs

A static IP is required on every node so the cluster stays stable. Follow these steps on each server.

### Step 1 — Disable cloud-init network control

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Add this line:

```
network: {config: disabled}
```

### Step 2 — Edit Netplan config

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace the contents with (adjust the IP for each server):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.145.154/24     # ← Change this per server
      routes:
        - to: default
          via: 192.168.145.2     # ← Your gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

### Step 3 — Apply and verify

```bash
sudo netplan generate
sudo netplan apply

# Verify
ip a show ens33
ip route
ping -c 3 192.168.145.2
ping -c 3 8.8.8.8
ping -c 3 google.com
```

---

## 3. Open Required Firewall Ports

Run this on the **Wazuh Server** before installation:

```bash
sudo ufw enable

sudo ufw allow 1514/tcp   # Agent communication
sudo ufw allow 1515/tcp   # Agent enrollment
sudo ufw allow 1516/tcp   # Cluster
sudo ufw allow 55000/tcp  # Wazuh API
sudo ufw allow 9200/tcp   # Indexer REST API
sudo ufw allow 9300:9400/tcp  # Indexer cluster
sudo ufw allow 443/tcp    # Dashboard (HTTPS)

sudo ufw reload
sudo ufw status verbose
```

---

## 4. Install Wazuh Indexer

### Step 1 — Download installer and config

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

### Step 2 — Edit config.yml

```bash
nano ./config.yml
```

Set your actual server IPs:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "<indexer-node-ip>"      # e.g. 192.168.145.160

  server:
    - name: wazuh-1
      ip: "<wazuh-manager-ip>"     # Same machine in single-node setup

  dashboard:
    - name: dashboard
      ip: "<dashboard-node-ip>"    # Same machine in single-node setup
```

### Step 3 — Generate config files

```bash
bash wazuh-install.sh --generate-config-files
```

### Step 4 — Install the indexer node

```bash
bash wazuh-install.sh --wazuh-indexer node-1
```

### Step 5 — Initialize the cluster

```bash
bash wazuh-install.sh --start-cluster
```

### Step 6 — Verify the cluster

Get the admin password:

```bash
tar -axf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O | grep -P "\'admin\'" -A 1
```

Test the indexer (use the password from above when prompted):

```bash
curl -k -u admin https://<WAZUH_INDEXER_IP>:9200
curl -k -u admin https://<WAZUH_INDEXER_IP>:9200/_cat/nodes?v
```

### Step 7 — Disable auto-updates (recommended)

```bash
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
```

---

## 5. Install Wazuh Server (Manager)

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
bash wazuh-install.sh --wazuh-server wazuh-1
```

### ⚠️ If you get "Unable to locate package wazuh-manager"

Run these steps to manually add the repository:

```bash
# Step 1 — Install dependencies
sudo apt update
sudo apt install -y curl apt-transport-https lsb-release gnupg

# Step 2 — Add the GPG key and repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | sudo gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh-archive-keyring.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update

# Step 3 — Verify the package is available
apt-cache policy wazuh-manager
```

You should see output like:

```
wazuh-manager:
  Installed: (none)
  Candidate: 4.14.2-1
```

Now run the installer again:

```bash
bash wazuh-install.sh --wazuh-server wazuh-1
```

### Disable auto-updates

```bash
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
```

---

## 6. Install Wazuh Dashboard

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
bash wazuh-install.sh --wazuh-dashboard dashboard
```

> If it fails, run the same command again — it usually succeeds on the second attempt.

### Get all passwords

```bash
tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

### Disable auto-updates

```bash
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
```

### ✅ Access the Dashboard

Open your browser and go to:

```
https://192.168.145.160
```

Login with `admin` and the password from the passwords file.

---

## 7. Add Agents

### Ubuntu/Linux Agent

1. In the dashboard, click **Deploy new agent**
2. Select package: **DEB amd64**
3. Enter your Wazuh server IP
4. Give the agent a name
5. Copy and run the generated command on the agent machine, for example:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.2-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.145.160' \
     WAZUH_AGENT_GROUP='default' \
     WAZUH_AGENT_NAME='ubuntu-agent-1' \
     dpkg -i ./wazuh-agent_4.14.2-1_amd64.deb
```

6. Start the agent:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

### Windows Agent

1. In the dashboard, click **Deploy new agent**
2. Select package: **MSI 32/64 bits**
3. Enter server IP and agent name
4. Copy the command and run it in **PowerShell as Administrator**

**If the agent isn't showing on the dashboard:**

```powershell
# Check the config file
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
# Verify the Wazuh server IP is correct

# Restart the service
net stop wazuh
net start wazuh

# Or via PowerShell
Restart-Service wazuh
```

---

## 8. File Integrity Monitoring (FIM)

By default, Wazuh does not generate FIM alerts. You need to configure which directories to monitor.

On the **Linux agent**, edit the config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<syscheck>` section and add this line inside it:

```xml
<directories check_all="yes" report_changes="yes" realtime="yes">/root</directories>
```

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

Now any file created, modified, or deleted in `/root` will trigger an alert on the dashboard.

---

## 9. Generate Test Alerts

Use these commands to generate different types of alerts and verify your setup is working.

### On the Linux Agent

```bash
# 1. SSH brute-force / failed login simulation
ssh wronguser@localhost

# 2. Successful sudo command
sudo ls /root

# 3. FIM alerts — create, modify, and delete files
sudo touch /etc/fim_test.txt
sudo nano /etc/fim_test.txt    # Add some content and save
sudo rm /etc/fim_test.txt

# 4. FIM in /root directory
cd /root
echo "test" > testfile.txt
echo "modified" >> testfile.txt
rm testfile.txt
```

### On the Windows Agent (PowerShell as Admin)

```powershell
# FIM alerts — file created, modified, deleted
New-Item C:\FIM-Test\test1.txt
Add-Content C:\FIM-Test\test1.txt "hello"
Remove-Item C:\FIM-Test\test1.txt
```

> **Tip:** If alerts aren't appearing on the dashboard, check Filebeat:
> ```bash
> systemctl status filebeat
> systemctl restart filebeat   # if inactive
> ```

---

## 10. Apache + ModSecurity WAF

### Install Apache

On the **agent** server:

```bash
sudo apt install apache2 -y
```

Verify it's running: `http://192.168.145.158`

### Install ModSecurity

ModSecurity is an application-layer firewall that inspects HTTP traffic inside Apache.

```bash
sudo apt install libapache2-mod-security2 -y
sudo a2enmod security2
sudo mkdir -p /etc/modsecurity/rules
```

### Enable OWASP Core Rule Set (CRS)

```bash
cd /etc/modsecurity
sudo cp modsecurity.conf-recommended modsecurity.conf
sudo ln -s /usr/share/modsecurity-crs/rules/*.conf /etc/modsecurity/rules/
```

Edit the config to turn on enforcement mode:

```bash
sudo nano /etc/modsecurity/modsecurity.conf
```

Find and change:

```
SecRuleEngine DetectionOnly
```

To:

```
SecRuleEngine On
```

```bash
sudo systemctl restart apache2
```

### Attack Simulations

Test that ModSecurity is blocking attacks:

```bash
# SQL Injection
curl "http://192.168.145.158/?id=1%27%20OR%20%271%27=%271"

# XSS (Cross-Site Scripting)
curl "http://192.168.145.158/?test=%3Cscript%3Ealert(1)%3C/script%3E"

# Directory Traversal
curl "http://192.168.145.158/?file=../../../../etc/passwd"

# Shellshock (from Kali Linux)
curl -H 'User-Agent: () { :; }; echo Shellshock-Test' http://192.168.145.158/
```

Or open these in a browser:

```
http://192.168.145.158/?id=1%27%20OR%20%271%27=%271
http://192.168.145.158/?q=%3Cscript%3Ealert(1)%3C/script%3E
http://192.168.145.158/?file=../../../../etc/passwd
```

---

## 11. Suricata IDS Integration

Suricata is a network intrusion detection system. Integrating it with Wazuh lets you see network-level alerts alongside host-based alerts.

### Step 1 — Install Suricata

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install suricata -y
```

### Step 2 — Download Emerging Threats rules

```bash
cd /tmp/
curl -LO https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz
sudo mkdir /etc/suricata/rules
sudo mv rules/*.rules /etc/suricata/rules/
sudo chmod 640 /etc/suricata/rules/*.rules
```

### Step 3 — Configure Suricata

```bash
sudo nano /etc/suricata/suricata.yaml
```

Set these values:

```yaml
vars:
  address-groups:
    HOME_NET: "192.168.145.0/24"   # ← Your network
    EXTERNAL_NET: "any"

default-rule-path: /etc/suricata/rules
rule-files:
  - "*.rules"

stats:
  enabled: yes

af-packet:
  - interface: ens33    # ← Your network interface (check with: ip a)
```

### Step 4 — Validate the config

```bash
# Fix permissions
sudo chown -R suricata:suricata /etc/suricata/rules
sudo chmod 640 /etc/suricata/rules/*.rules

# Test configuration
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

You should see: `Configuration provided was successfully loaded`

### Step 5 — Start Suricata

```bash
sudo systemctl restart suricata
```

### Step 6 — Connect Suricata logs to Wazuh

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this block at the end (before `</ossec_config>`):

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

```bash
sudo systemctl restart wazuh-agent
```

### Attack Simulations for Suricata

From the **Wazuh manager server**:

```bash
ping -c 20 <AGENT_IP>
```

From **Kali Linux** (or any machine):

```bash
nmap -sS <AGENT_IP>
nmap -p- <AGENT_IP>
```

Watch alerts appear on the Wazuh dashboard in real time.

---

## 12. Enable Archive Logs

Archive logs let you see **all events**, not just alerts.

### Step 1 — Enable in ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find:

```xml
<logall>no</logall>
<logall_json>no</logall_json>
```

Change both to `yes`.

### Step 2 — Enable in Filebeat

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Find the archives section and set `enabled: true`.

### Step 3 — Restart services

```bash
sudo systemctl restart wazuh-manager
sudo systemctl restart filebeat
```

### Step 4 — Add index pattern in Dashboard

1. Go to the Wazuh dashboard
2. Navigate to **Indexer management → Index Patterns**
3. Add `wazuh-archives-*` as a new index pattern
4. The archive view will appear in the top-right of the dashboard

---

## 13. Backup & Upgrade

### Check installed versions

```bash
sudo apt list --installed wazuh-indexer
sudo apt list --installed wazuh-manager
sudo apt list --installed wazuh-dashboard
```

### Create a snapshot repository (do this BEFORE upgrading)

```bash
# Step 1 — Edit indexer config
sudo nano /etc/wazuh-indexer/opensearch.yml

# Add this line anywhere in the file:
path.repo: ["/mnt/snapshots"]

# Step 2 — Create and permission the folder
sudo mkdir -p /mnt/snapshots
sudo chown -R wazuh-indexer:wazuh-indexer /mnt/snapshots
sudo chmod 750 /mnt/snapshots

# Step 3 — Restart indexer
sudo systemctl restart wazuh-indexer
```

### Create a repository in the Dashboard

1. Go to **Indexer management → Snapshot Management → Repositories**
2. Create a new repository pointing to `/mnt/snapshots`

### Take a snapshot

1. Go to **Indexer management → Snapshot Management → Snapshots**
2. Click **Take snapshot**
3. Enter a name and select the repository
4. Enable **Include cluster state in snapshots**
5. Click **Add**

> Snapshots are saved to `/mnt/snapshots` on disk. You can also automate this with a cron job and configure email notifications when a snapshot completes.

### Upgrade

Follow the [official Wazuh upgrade documentation](https://documentation.wazuh.com/current/upgrade-guide/index.html) after taking your snapshot.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Unable to locate package wazuh-manager` | Manually add the Wazuh repo — see [Step 5](#-if-you-get-unable-to-locate-package-wazuh-manager) |
| Agents not showing on dashboard | Run `Restart-Service wazuh` (Windows) or `systemctl restart wazuh-agent` (Linux) |
| No alerts on dashboard | Check `systemctl status filebeat` and restart if inactive |
| Windows agent wrong IP | Edit `C:\Program Files (x86)\ossec-agent\ossec.conf` and restart the service |
| Dashboard install fails | Run the install command a second time |
| Suricata rules not loading | Check permissions with `sudo chown -R suricata:suricata /etc/suricata/rules` |

---

## 📌 Quick Reference — Key IPs & Ports

| Service | Default Port | Notes |
|---|---|---|
| Wazuh Dashboard | 443 | `https://<server-ip>` |
| Wazuh API | 55000 | REST API |
| Wazuh Indexer | 9200 | OpenSearch REST |
| Agent enrollment | 1515 | New agents register here |
| Agent communication | 1514 | Ongoing log shipping |

---

## 📁 Project Structure

```
wazuh-setup/
├── README.md              ← You are here
├── config/
│   └── config.yml         ← Example Wazuh cluster config
├── agents/
│   ├── ubuntu-agent/
│   │   └── ossec.conf     ← Example agent config with FIM
│   └── windows-agent/
│       └── ossec.conf     ← Example Windows agent config
└── suricata/
    └── suricata.yaml      ← Example Suricata config
```

---

## 📚 References

- [Wazuh Official Documentation](https://documentation.wazuh.com/)
- [Wazuh Installation Guide](https://documentation.wazuh.com/current/installation-guide/index.html)
- [Emerging Threats Rules](https://rules.emergingthreats.net/)
- [OWASP ModSecurity CRS](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [Suricata Documentation](https://suricata.readthedocs.io/)

---

> ⭐ If this helped you, consider starring the repo!
