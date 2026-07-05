# Suricata NIDS Integration with Wazuh XDR/SIEM

![Wazuh](https://img.shields.io/badge/Wazuh-4.14-blue) ![Suricata](https://img.shields.io/badge/Suricata-8.0.5-orange) ![Kali Linux](https://img.shields.io/badge/Kali_Linux-Agent-red) ![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-green) ![Status](https://img.shields.io/badge/Status-Active-brightgreen)

## Overview

This project documents the integration of **Suricata 8.0.5** (Network Intrusion Detection System) into an existing Wazuh XDR/SIEM home lab environment. Suricata is deployed on a Kali Linux endpoint and configured to forward network alerts to the Wazuh Manager in real time via the Wazuh agent.

This is Part 2 of my Home SOC Lab series. Part 1 covers the full Wazuh multi-endpoint deployment with Windows 10 and Kali Linux agents.

---

## Lab Environment

| Component | Details |
|---|---|
| Wazuh Manager | Ubuntu 24.04 — 192.168.4.55 |
| Kali Linux Agent | Kali Linux — 192.168.4.30 |
| Windows 10 Agent | Enrolled in Wazuh Manager |
| Suricata Version | 8.0.5 |
| Wazuh Version | 4.14 |
| Hypervisor | Oracle VirtualBox |
| Rules Loaded | 401 Emerging Threats rules |

---

## Architecture

```
[Network Traffic]
       |
       v
[Suricata NIDS on Kali Linux - eth0]
       |
       v
[/var/log/suricata/eve.json]
       |
       v
[Wazuh Agent on Kali Linux]
       |
       v
[Wazuh Manager on Ubuntu 24.04]
       |
       v
[Wazuh Dashboard - Alerts Visible]
```

---

## What Was Implemented

- Suricata 8.0.5 installed on Kali Linux via OISF stable PPA
- Emerging Threats (ET) Open ruleset downloaded and configured (401 rules loaded)
- `suricata.yaml` configured with correct HOME_NET, interface (eth0), stats, and rule path
- Suricata running as a systemd service — enabled on boot
- Wazuh agent `ossec.conf` updated to read `/var/log/suricata/eve.json`
- End-to-end pipeline verified — 38 Suricata alerts visible in Wazuh Discover

---

## Step-by-Step Implementation

### Step 1 — Install Suricata on Kali Linux

![Wazuh official documentation showing install commands](WazuhIDS/Wazuh%20IDS.JPG)

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install suricata -y
```

Verify installation:

```bash
suricata --version
```

In my case this confirmed **Suricata version 8.0.5**.

---

### Step 2 — Download Emerging Threats Ruleset

```bash
cd /tmp && curl -LO https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz && sudo mkdir /etc/suricata/rules && sudo mv rules/*.rules /etc/suricata/rules/
sudo find /etc/suricata/rules -name "*.rules" -exec chmod 777 {} \;
```

---

### Step 3 — Configure suricata.yaml

![Wazuh official documentation showing required suricata.yaml changes](WazuhIDS/Wazuh%20IDS%201.JPG)

```bash
sudo nano /etc/suricata/suricata.yaml
```

#### 3.1 — Set HOME_NET

```yaml
HOME_NET: "[192.168.0.0/22]"
EXTERNAL_NET: "any"
```

![suricata.yaml showing HOME_NET and EXTERNAL_NET configured](WazuhIDS/Wazuh%20IDS%202.JPG)

#### 3.2 — Enable Stats

```yaml
stats:
  enabled: yes
```

![suricata.yaml showing stats enabled](WazuhIDS/Wazuh%20IDS%203.JPG)

#### 3.3 — Set Rule Path

```yaml
default-rule-path: /etc/suricata/rules
rule-files:
  - "*.rules"
```

![suricata.yaml showing default-rule-path and rule-files configured](WazuhIDS/Wazuh%20IDS%204.JPG)

#### 3.4 — Set Network Interface

```yaml
af-packet:
  - interface: eth0
```

![suricata.yaml showing af-packet interface set to eth0](WazuhIDS/Wazuh%20IDS%205.JPG)

> **Note:** Run `ip a` to confirm your interface name before editing. In this lab it was `eth0`.

Save and exit with `Ctrl+X`, then `Y`, then `Enter`.

---

### Step 4 — Start and Enable Suricata

![Wazuh official documentation showing restart Suricata command](WazuhIDS/Wazuh%20IDS%206.JPG)

```bash
sudo systemctl start suricata
sudo systemctl enable suricata
sudo systemctl status suricata
```

![Kali Linux terminal showing Suricata active and running](WazuhIDS/Wazuh%20IDS%207.JPG)

Expected output confirms:
- `Active: active (running)`
- Suricata running with: `/usr/bin/suricata -D --af-packet -c /etc/suricata/suricata.yaml`

---

### Step 5 — Configure Wazuh Agent to Read eve.json

![Wazuh official documentation showing ossec.conf localfile block](WazuhIDS/Wazuh%20IDS%208.JPG)

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add inside `<ossec_config>`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

![ossec.conf showing localfile block added with eve.json location highlighted](WazuhIDS/Wazuh%20IDS%209.JPG)

> **Important:** Ensure the XML block is properly aligned with surrounding configuration. Misaligned XML will cause the Wazuh agent to fail on restart.

Save the file.

---

### Step 6 — Restart Wazuh Agent

![Wazuh official documentation showing restart wazuh-agent command](WazuhIDS/Wazuh%20IDS%2010.JPG)

```bash
sudo systemctl restart wazuh-agent
```

---

### Step 7 — Verify eve.json Logging

```bash
# Check last entries
tail /var/log/suricata/eve.json

# Watch live events
tail -f /var/log/suricata/eve.json
```

![Kali Linux terminal showing tail command output with eve.json data](WazuhIDS/Wazuh%20IDS%2011.JPG)

Confirmed output includes `rules_loaded: 401` and active event types.

---

### Step 8 — Attack Emulation

Ping from Wazuh Manager (Ubuntu) toward Kali Linux to generate detectable traffic:

```bash
ping -c 20 192.168.4.30
```

![Ubuntu terminal showing ping command to Kali Linux IP with 20 packets sent](WazuhIDS/Wazuh%20IDS%2012.JPG)

> **Note:** Official Wazuh documentation performs this ping toward an Ubuntu endpoint. In this lab, Suricata is on Kali Linux — so the ping targets Kali's IP instead. Traffic crosses eth0 where Suricata is monitoring.

---

### Step 9 — Verify Alerts in Wazuh Dashboard

In Wazuh Discover, apply filter:

```
location: /var/log/suricata/eve.json
```

![Wazuh Discover dashboard showing 38 hits with eve.json location filter applied](WazuhIDS/Wazuh%20IDS%2013.JPG)

**Result: 38 hits confirmed** — Suricata alerts flowing from Kali Linux into Wazuh Manager in real time.

Alert fields visible in dashboard:
- `agent.name: kali`
- `data.event_type: alert`
- `data.alert.signature` — rule that triggered
- `data.src_ip` / `data.dest_ip`
- `data.in_iface: eth0`

---

## Key Findings

| Finding | Detail |
|---|---|
| Rules Loaded | 401 Emerging Threats rules |
| Rules Failed | 22 (minor, does not affect operation) |
| Alerts Generated | 38 confirmed in Wazuh Discover |
| Interface Monitored | eth0 |
| Log File | /var/log/suricata/eve.json |
| Pipeline Status | Fully operational |

---

## Lessons Learned

- **Official Wazuh docs assume Ubuntu** — adapt interface and attack emulation steps when deploying Suricata on Kali Linux
- **Verify at each stage** — check eve.json locally with `tail` before checking the Wazuh dashboard
- **YAML indentation is critical** — suricata.yaml is sensitive to spacing; misalignment breaks the configuration
- **ossec.conf alignment matters** — XML blocks must align with surrounding config or Wazuh agent fails to restart
- **IPv6 can silently bypass rules** — when testing with curl, use `-4` flag to force IPv4 so Suricata captures the traffic correctly

---

## Tools Used

| Tool | Purpose |
|---|---|
| Suricata 8.0.5 | Network Intrusion Detection System (NIDS) |
| Wazuh 4.14 | XDR/SIEM — log aggregation and alerting |
| Emerging Threats Ruleset | Suricata detection signatures |
| Oracle VirtualBox | Hypervisor |
| Kali Linux | Suricata endpoint / Wazuh agent |
| Ubuntu 24.04 | Wazuh Manager |

---

## Related Articles

- [How I Added Real-Time Threat Detection to My Home SOC Using Suricata and Wazuh](https://medium.com/@Nehachoudhary) — Published on System Weakness (Medium)
- Part 1: Building a Multi-Endpoint XDR/SIEM Lab with Wazuh — Published on System Weakness (Medium)

---

## Author

**Kuldeep Choudhary**
- LinkedIn: [linkedin.com/in/kuldeep-choudhary-28193311b](https://linkedin.com/in/kuldeep-choudhary-28193311b)
- GitHub: [github.com/Deep2511](https://github.com/Deep2511)
- Medium: System Weakness Publication

---

*All work performed in a controlled home lab environment for educational purposes.*
