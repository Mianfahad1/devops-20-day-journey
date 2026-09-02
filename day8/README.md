# Day 8 — AWS S3, CLI & Cloud Automation

## Lectures Completed: 32, 34, 35, 37, 38

### Key Achievements:
- Installed and configured AWS CLI
- Created S3 bucket and made a file publicly accessible
- Wrote a Bash script to launch EC2 instances using AWS CLI
- Launched a new EC2 instance in a private subnet
- Created and attached an IAM role with S3 read-only permissions
- Verified role-based access (least privilege principle)
- Set up EventBridge Scheduler to auto-stop EC2 after 2 hours

### Commands Practiced:
```bash
aws s3 ls
aws s3 cp
aws ec2 run-instances
aws iam create-role
aws iam attach-role-policy
aws ec2 associate-iam-instance-profile
