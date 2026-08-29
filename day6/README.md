# 🚀 Day 6: Linux Administration, Storage & Networking Mastery

**Date:** 2026-08-29  
**Environment:** AWS EC2 (Ubuntu 24.04 LTS)  
**Role Simulation:** Junior DevOps / Cloud Administrator at a SaaS company

---

## 📌 Executive Summary

Today I built a rock-solid foundation in Linux system administration by simulating real-world production tasks. I went beyond basic commands and implemented persistent storage, automated log rotation, and a hardened firewall. This lab directly translates to keeping production web servers secure, scalable, and recoverable.

**Key Achievements:**
- ✅ Architected a persistent EBS volume that survives reboots (critical for database/log storage).
- ✅ Hardened the server using UFW (blocking all ports except SSH & HTTP).
- ✅ Implemented `logrotate` to prevent logs from crashing production servers.
- ✅ Executed advanced text processing (`grep`, `awk`, `cut`) for security incident response.
- ✅ Mastered file system links and system performance stress-testing.

---

## 🛠️ Technologies & Tools Used
- **Cloud:** AWS EC2, AWS EBS (Elastic Block Store)
- **OS:** Ubuntu 24.04 LTS
- **Storage:** `fdisk`, `mkfs.ext4`, `/etc/fstab`
- **Security:** UFW (Uncomplicated Firewall)
- **Logging:** `logrotate`, `grep`, `awk`
- **Networking:** `ss`, `netstat`, `traceroute`, `nslookup`

---

## 📂 Lab Breakdown & Execution

### 1. Backup & Archiving (Lecture 18)
- **Tools:** `tar`, `gzip`, `bzip2`, `zip`
- **Action:** Built a structured directory (`~/backup-assignment`), compressed files individually, and created full archives (`tar.gz`, `tar.bz2`).
- **Outcome:** Successfully created portable, compressed backups ready for offloading to S3.

### 2. File System Links (Lecture 19)
- **Soft Link:** Created a symbolic link (`ln -s`). When the original was deleted, the link **broke** (`No such file or directory`).
- **Hard Link:** Created a hard link (`ln`). Even after deleting the original, the hard link retained the data (identical **inode** numbers).

### 3. System Performance & Process Management (Lecture 20)
- **Tools:** `top`, `htop`, `vmstat`, `ps`, `kill`
- **Stress Test:** Spawned 4 `yes > /dev/null` processes to peg the CPU at 100%. Used `vmstat` to observe CPU spike and kill processes cleanly.
- **Service Management:** Installed NGINX and practiced `systemctl` lifecycle (`start`, `stop`, `restart`, `status`, `enable`).

### 4. EBS Persistent Storage (Lecture 21) — The Crown Jewel
- **Attachment:** Created a 5GB EBS volume in AWS Console (`us-east-1a`) and attached as `/dev/nvme1n1`.
- **Partitioning:** Used `sudo fdisk /dev/nvme1n1` to create `nvme1n1p1`.
- **Formatting:** Formatted with `sudo mkfs.ext4 /dev/nvme1n1p1`.
- **Mounting:** Mounted to `/mnt/data` and verified via `df -h`.
- **Persistence:** Extracted UUID using `blkid` and added to `/etc/fstab`.
  - *Troubleshooting Wins:*
    - Fixed critical typo (`UID` → `UUID`) in `/etc/fstab`.
    - Resolved `Permission denied` by running `sudo chown -R ubuntu:ubuntu /mnt/data`.
  - **Result:** After reboot, `df -h` confirmed volume automatically remounted with data intact!

### 5. Log Management & Security Auditing (Lecture 22)
- **Data:** Parsed a dummy `auth.log` file.
- **Parsing:** Used `grep` to filter SSH logs and `awk` to extract timestamps and usernames.
- **Automation:** Configured `logrotate` for `/opt/log_reports/ssh_access.log` (size 10k, rotate 4, compress).

### 6. Networking & Firewall Hardening (Lecture 23)
- **DNS & Routing:** Verified connectivity using `ping`, `traceroute`, and `nslookup`.
- **Port Analysis:** Used `ss -tuln` and `netstat -tulnp` to audit open ports.
- **Firewall:** Installed and enabled **UFW** with rules: `allow ssh`, `allow http`.

---

## 📊 Verification Outputs

### Persistent Mount After Reboot
```bash
ubuntu@ip-10-20-1-77:~$ df -h | grep /mnt/data
/dev/nvme1n1p1   4.9G   24K  4.6G   1% /mnt/data

-- UFW Firewall Status

ubuntu@ip-10-20-1-77:~$ sudo ufw status verbose
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere 

  EBS Write Test


ubuntu@ip-10-20-1-77:/mnt/data$ echo "EBS persistent storage works!" > my_projects/test.txt
ubuntu@ip-10-20-1-77:/mnt/data$ cat my_projects/test.txt
EBS persistent storage works!

🧠 Key Takeaways for My Career
Storage is not just "space"; it's a responsibility. Using /etc/fstab for persistence and chown for permissions is how real cloud engineers prevent data loss during deployments.

Security is layered. UFW acts as the network gatekeeper, while Linux file permissions act as the OS gatekeeper.

Troubleshooting mindset matters. The Permission denied and UID vs UUID errors taught me that the devil is in the details.

Monitoring saves money. vmstat and logrotate are free tools that prevent costly server crashes.

Prepared by: Fahad Ali Seemab
Course: 20-Day DevOps Engineering Journey (Mise Academy)
Status: Day 6 Completed ✅
