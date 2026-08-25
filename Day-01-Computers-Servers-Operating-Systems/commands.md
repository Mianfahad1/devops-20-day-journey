# 🛠️ Day 1 — Linux Commands Practiced

## 👤 User & System Information

whoami
hostname
uname -a
cat /etc/os-release

## 💻 CPU, Memory & Storage

nproc
free -h
df -h
lsblk
uptime

## ⚙️ Processes & Services

ps aux
systemctl --failed
systemctl list-units --type=service --state=running --no-pager
systemctl status wazuh-manager --no-pager
systemctl status elasticsearch --no-pager
systemctl status logstash --no-pager
systemctl status wazuh-dashboard --no-pager
systemctl is-enabled wazuh-manager
systemctl is-enabled elasticsearch

## 📁 Files & Directories

pwd
mkdir -p ~/day1-lab
cd ~/day1-lab
touch test.txt
ls -l test.txt
echo "DevOps Day 1" > test.txt
cat test.txt

## 🔐 File Permissions

chmod 600 test.txt
ls -l test.txt

## 🔑 SSH

systemctl status ssh --no-pager
ss -tlnp | grep :22
