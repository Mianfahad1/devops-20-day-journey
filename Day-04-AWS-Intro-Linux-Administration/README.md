# Day 4 — AWS Intro & Linux Administration I
Phase 1: Foundations & Linux Mastery · Day 4 of 20

### 📚 Lectures Covered (5)
- `Day 3_Intro to Cloud Computing` (20 slides)
- `Day 12 - AWS Intro & Server Creation` (18 slides)
- `Day 13_Deploy Web App on AWS` (13 slides)
- `Day 14 Basic Linux Commands` (18 slides)
- `Day 15 File System User and Group Management` (16 slides)

### 💻 Lab Host & Environment
- **Host:** Ubuntu 24.04 LTS on AWS EC2 (`m7i-flex.large`, `us-east-1a`)
- **Web Server:** Nginx 1.24.0 (HTTP Port 80)
- **Security & SIEM:** Wazuh Dashboard (HTTPS Port 443) + ELK Stack

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

### 1 · Cloud Service Models & The Shared Responsibility Matrix

* **IaaS (e.g. AWS EC2):** AWS manages physical data centers, host hardware, and Nitro hypervisors. You manage the guest OS, patches, firewall security groups, and applications.
* **PaaS (e.g. Elastic Beanstalk):** Cloud provider manages the OS and runtime; you only provide code and configuration.
* **SaaS (e.g. Gmail / GitHub):** Provider manages the entire stack; you consume the service.
* **IaaS Reality:** Running an EC2 instance means AWS handles physical infrastructure. Everything inside the instance—from patching vulnerabilities to securing open ports—is entirely our responsibility.

---

### 2 · EC2 Provisioning & Nginx Web Server Deployment

```bash
sudo apt-get update -y
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
sudo ss -tlnp | grep :80	
