# Day 7 — Text Processing & File Searching

## Lectures Completed: 24, 25, 26, 27, 28

### Key Skills Acquired:

**awk:**
- Column extraction (`$1`, `$3`)
- Conditional filtering (`$2 > 150`)
- `BEGIN` and `END` blocks
- Sum and average calculations
- Log validation with `NF`
- Clean output with `gsub`
- Filtering with AND conditions (`$3=="dev" && $2>120`)

**sed:**
- Find and replace (`s/old/new/g`)
- In-place editing (`-i`)
- Delete lines (`/pattern/d`)

**grep:**
- Case-insensitive search (`-i`)
- Line numbers (`-n`)
- Invert match (`-v`)
- Regular expressions (`-E`)
- Count occurrences (`-c`)
- Process filtering (`ps aux | grep`)
- Show matched part (`-o`)
- Highlight results (`--color`)

**cut:**
- Single and multiple field extraction (`-f1,7`)
- Custom delimiters (`-d ','`)
- Combine with `grep` for filtering
- CSV parsing for cloud inventory (EC2, IAM, users)

**find & locate:**
- Search by name (`-name`)
- Search by type (`-type f`)
- Search by modification time (`-mtime -7`)
- System-wide search with `locate`
- Update locate database (`updatedb`)

### Sample Commands Practiced:
```bash
awk '$2 > 150 { print $1, $2 }' sample.txt
grep -E "403|404" access.log
cut -d ',' -f1,6 ec2_instances.csv
grep "Failed password" auth.log | cut -d ' ' -f11
find /opt/devops-app/configs -name "*.txt"
locate backup.gz.tar


