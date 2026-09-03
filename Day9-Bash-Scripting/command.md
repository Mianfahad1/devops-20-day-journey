# Essential Bash Commands for Cloud Automation

This file documents the core commands used during Day 9's Bash Scripting tasks for Cloud Administration.

## 🛠️ File & Script Management
- `nano <filename>.sh` - Opens the text editor to create or edit a script.
- `cat > <filename>.sh << 'EOF'` - Creates a file and writes content directly into it.
- `bash <filename>.sh` - Executes a shell script.
- `sudo bash <filename>.sh` - Executes a script with root privileges (e.g., for backups).
- `ls -l` - Lists files with detailed permissions.
- `rm -rf <directory>` - Forcefully removes a directory and its contents.

## 💻 Variables, Inputs & Outputs
- `read variable_name` - Takes user input and stores it in a variable.
- `echo "text $variable"` - Prints text and variable values to the terminal.
- `$1`, `$2` - Positional arguments passed to a script from the command line.
- `$(command)` - Captures the output of a command into a variable.

## 👥 User Management (Automation)
- `sudo useradd -m <username>` - Creates a new user with a home directory.
- `sudo passwd <username>` - Interactively sets a user's password.
- `echo "$USERNAME:$PASSWORD" | sudo chpasswd` - Sets a user's password non-interactively.

## 💾 Backup & System Monitoring
- `mkdir -p <directory>` - Creates a directory (and parent directories) if it doesn't exist.
- `cp -r <source>/* <destination>` - Copies all files recursively to a backup directory.
- `tar -czf <backup_name>.tar.gz /etc` - Creates a compressed backup archive.
- `du -sh <directory>` - Displays the total disk usage of a directory.
- `df / | grep / | awk '{ print $5 }' | sed 's/%//g'` - Extracts the percentage of disk usage.
