# 🛠️ Day 1 — Linux Commands Practiced

These are the Linux commands I practiced during Day 1 of my DevOps Engineering Journey.

## 👤 User & System Information

Check current user:
    whoami

Check hostname:
    hostname

Check kernel information:
    uname -a

Check operating system:
    cat /etc/os-release

## 💻 CPU, Memory & Storage

Check CPU count:
    nproc

Check memory and swap:
    free -h

Check disk usage:
    df -h

Check block devices:
    lsblk

Check system uptime and load:
    uptime

## ⚙️ Processes & Services

View running processes:
    ps aux

Check failed services:
    systemctl --failed

List running services:
    systemctl list-units --type=service --state=running --no-pager

Check Wazuh Manager:
    systemctl status wazuh-manager --no-pager

Check Elasticsearch:
    systemctl status elasticsearch --no-pager

Check Logstash:
    systemctl status logstash --no-pager

Check Wazuh Dashboard:
    systemctl status wazuh-dashboard --no-pager

Check Wazuh Manager startup:
    systemctl is-enabled wazuh-manager

Check Elasticsearch startup:
    systemctl is-enabled elasticsearch

## 📁 Files & Directories

Check current directory:
    pwd

Create Day 1 lab directory:
    mkdir -p ~/day1-lab

Enter the lab directory:
    cd ~/day1-lab

Create a test file:
    touch test.txt

Check file details:
    ls -l test.txt

Write text to the file:
    echo "DevOps Day 1" > test.txt

Read the file:
    cat test.txt

## 🔐 File Permissions

Change file permissions:
    chmod 600 test.txt

Verify permissions:
    ls -l test.txt

## 🔑 SSH

Check SSH service:
    systemctl status ssh --no-pager

Check SSH port 22:
    ss -tlnp | grep :22

## 🧠 Day 1 Summary

Practiced Linux system identification, CPU and memory inspection, storage management, process monitoring, systemd service management, file operations, Linux permissions, and SSH troubleshooting on a real AWS EC2 Ubuntu server.
