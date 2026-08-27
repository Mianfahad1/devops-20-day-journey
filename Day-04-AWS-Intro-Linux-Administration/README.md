# Day 4 — AWS Intro & Linux Administration I

> **Phase 1: Foundations & Linux Mastery · Day 4 of 20**

---

## 📚 Lectures Covered (5)

- **Day 3** — Intro to Cloud Computing *(20 slides)*
- **Day 12** — AWS Intro & Server Creation *(18 slides)*
- **Day 13** — Deploy Web App on AWS *(13 slides)*
- **Day 14** — Basic Linux Commands *(18 slides)*
- **Day 15** — File System User and Group Management *(16 slides)*

---

## 💻 Lab Host & Environment

| Component | Details |
| :--- | :--- |
| **Host** | Ubuntu 24.04 LTS on AWS EC2 (`m7i-flex.large`, `us-east-1a`) |
| **Web Server** | Nginx 1.24.0 (HTTP Port 80) |
| **Security & SIEM** | Wazuh Dashboard (HTTPS Port 443) + ELK Stack |

---

## 📑 Contents

1. [Cloud Service Models & The Shared Responsibility Matrix](#1--cloud-service-models--the-shared-responsibility-matrix)
2. [EC2 Provisioning & Nginx Web Server Deployment](#2--ec2-provisioning--nginx-web-server-deployment)
3. [The Port 80 vs 443 Browser Routing Observation](#3--the-port-80-vs-443-browser-routing-observation)
4. [Live Access Logs: 200 OK vs 304 Not Modified](#4--live-access-logs-200-ok-vs-304-not-modified)
5. [Linux File System Hierarchy & Path Navigation](#5--linux-file-system-hierarchy--path-navigation)
6. [User & Group Management: Human vs Daemon Accounts](#6--user--group-management-human-vs-daemon-accounts)
7. [Privilege Escalation & Sudo Authorization](#7--privilege-escalation--sudo-authorization)
8. [Key Takeaways](#8--key-takeaways)

---

## 1 · Cloud Service Models & The Shared Responsibility Matrix

- **IaaS (e.g. AWS EC2)**: AWS manages physical data centers, host hardware, and Nitro hypervisors. You manage the guest OS, security patches, network security groups, and applications.
- **PaaS (e.g. AWS Elastic Beanstalk)**: Cloud provider manages the OS and runtime; you only provide code and configuration.
- **SaaS (e.g. Gmail / GitHub)**: Provider manages the entire stack; you consume the service.

> 💡 **IaaS Reality**: Running an EC2 instance means AWS handles physical infrastructure. Everything inside the instance—from patching vulnerabilities to securing open ports—is entirely our responsibility.

---

## 2 · EC2 Provisioning & Nginx Web Server Deployment

```bash
sudo apt-get update -y
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
sudo ss -tlnp | grep :80
```

**Output:**
```text
LISTEN 0 511 0.0.0.0:80 0.0.0.0:* users:(("nginx",pid=4231,fd=6),("nginx",pid=4230,fd=6))
```

- Nginx master process runs as `root` and worker processes run under `www-data`, bound to `0.0.0.0:80`.
- Deployed responsive landing page to `/var/www/html/index.html`.

---

## 3 · The Port 80 vs 443 Browser Routing Observation

When visiting the server's public IP in a browser without specifying the protocol, the Wazuh Dashboard login loaded instead of the Nginx page.

### Root Cause
- **Port 443 (HTTPS)**: `wazuh-dashboard` (Node.js) is listening.
- **Port 80 (HTTP)**: `nginx` is listening.

Modern browsers automatically upgrade naked URLs to `https://`. Navigating explicitly to `http://<public-ip>` routed directly to Port 80, immediately rendering the deployed web app.

---

## 4 · Live Access Logs: 200 OK vs 304 Not Modified

Inspecting `/var/log/nginx/access.log` while interacting with the server:

```bash
sudo tail -n 5 /var/log/nginx/access.log
```

**Live Output:**
```text
153.117.125.8 - - [27/Aug/2026:20:09:00 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0..."
153.117.125.8 - - [27/Aug/2026:20:10:42 +0000] "GET / HTTP/1.1" 200 734 "-" "Mozilla/5.0..."
::1 - - [27/Aug/2026:20:11:31 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.5.0"
```

| Log Entry | Status | Meaning |
| :--- | :---: | :--- |
| `GET /` | `304` | **Browser Caching**: Browser validated ETag; server confirmed zero changes. 0 bytes payload sent. |
| `GET /` | `200` | **Hard Refresh**: Fresh HTML payload (734 bytes) transmitted. |
| `HEAD /` | `200` | **`curl -I` request**: Nginx only returned headers; body omitted. |

---

## 5 · Linux File System Hierarchy & Path Navigation

Tested absolute vs relative path traversals across key directories:
- `/var/log/nginx/` (Application logs, owned by `www-data:adm`)
- `/etc/` (Configuration files and user databases)

> 📌 **Path Resolution**: Absolute paths always anchor to `/`; relative paths resolve from `$PWD`.

---

## 6 · User & Group Management: Human vs Daemon Accounts

```bash
sudo groupadd devops_team
sudo useradd -m -s /bin/bash john_dev
sudo passwd john_dev
sudo usermod -aG sudo,devops_team john_dev
```

### Verification
```bash
id john_dev
getent group devops_team
```

**Output:**
```text
uid=1001(john_dev) gid=1002(john_dev) groups=1002(john_dev),27(sudo),1001(devops_team)
devops_team:x:1001:john_dev
```

### Critical Security Insight in `/etc/passwd`
```text
wazuh:x:111:113::/var/ossec:/sbin/nologin
elasticsearch:x:113:115::/nonexistent:/bin/false
john_dev:x:1001:1002::/home/john_dev:/bin/bash
```

- **System / Daemon Accounts**: UID < 1000, shells set to `/sbin/nologin` or `/bin/false`. They run background services with zero interactive terminal access.
- **Human Users**: UID ≥ 1000, assigned a valid home directory (`/home/john_dev`) and interactive shell (`/bin/bash`).

---

## 7 · Privilege Escalation & Sudo Authorization

```bash
sudo -i -u john_dev
whoami      # Output: john_dev
pwd         # Output: /home/john_dev
sudo whoami # Output: root
```

`john_dev` authenticated with his own password and executed root commands via membership in group `27(sudo)` defined in `/etc/sudoers`.

---

## 8 · Key Takeaways

1. **Shared Responsibility in IaaS**: Hardware reliability is AWS’s job; OS configuration, service ports, and user authorization are entirely ours.
2. **Port Collisions & Protocol Upgrades**: A server hosting multiple services must be tested by protocol (`http://` vs `https://`) and port binding (`ss -tlnp`).
3. **HTTP 304 vs 200**: Web performance depends on caching headers. A `304` means client-side cache hit; a `200` represents actual network data egress.
4. **Daemon Isolation**: Production services must never have login shells (`/sbin/nologin`).
5. **Least Privilege & Sudo**: Grant administrative privileges via group membership (`sudo` group) rather than sharing the root password directly.

---
---

### 🧭 Navigation
| [⬅️ Day 03: Git & GitHub](../Day-03-Git-GitHub/) | [🏠 Main Roadmap](../README.md) | [Day 05: SDLC & Bash ➡️](../Day-05-SDLC-Bash-Scripting/) |


