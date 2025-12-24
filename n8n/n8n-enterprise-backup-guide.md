# n8n Enterprise Backup Setup Guide

## Overview
This guide documents the complete enterprise backup strategy for n8n running on AWS EC2, implementing a robust 3-2-1 backup approach with automated EBS snapshots and S3 application-level backups.

---

## System Information
- **n8n Instance:** EC2 i-XXXXXXXXXXXXX
- **Volume:** vol-XXXXXXXXXXXXX (8 GiB)
- **Region:** your-region (e.g., ap-southeast-2)
- **Domain:** https://yourdomain.com
- **Data Location:** `/var/lib/docker/volumes/n8n_data/_data`

---

## Part 1: EBS Snapshot Backups (AWS Data Lifecycle Manager)

### 1.1 Tag Your EBS Volume

**Navigate to:** EC2 → Volumes

```bash
# Find your volume attached to your instance
# Add tag:
Key: Name
Value: n8n-yourname-tech
```

**Verification:**
- Volume shows tag `Name = n8n-yourname-tech`
- Volume state is "In-use"
- Attached to correct instance

### 1.2 Create Data Lifecycle Manager Policy

**Navigate to:** EC2 → Lifecycle Manager → Create lifecycle policy

**Settings:**
- **Policy type:** EBS snapshot policy
- **Target resource type:** Volume
- **Target resource tags:** `Name = n8n-yourname-tech`
- **Description:** `Daily automated backups of n8n production volume`
- **IAM role:** Default role (auto-created)

**Schedule Configuration:**
- **Schedule name:** `daily backup`
- **Frequency:** Daily
- **Every:** 24 hours
- **Starting at:** `16:00` UTC (12 AM Singapore time)
- **Retention type:** Count
- **Keep:** 7 snapshots

### 1.3 Verify DLM Policy

**Check:** EC2 → Lifecycle Manager → Your policy should show as "Enabled"

**First snapshot:** Will be created at next scheduled time (16:00 UTC)

**View snapshots:** EC2 → Snapshots

---

## Part 2: Application-Level S3 Backups

### 2.1 Install AWS CLI on EC2

```bash
# SSH into EC2 instance
ssh -i "your-key-pair.pem" ubuntu@your-ec2-instance.ap-southeast-2.compute.amazonaws.com

# Download and install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install -y unzip
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Clean up
rm -rf aws awscliv2.zip
```

**Expected output:** `aws-cli/2.x.x ...`

### 2.2 Create IAM User and Access Keys

**Navigate to:** IAM → Users → Create user

**User details:**
- **User name:** `n8n-backup-user`
- **Access type:** Programmatic access only

**Permissions:**
- Attach policies directly:
  - `AmazonS3FullAccess`
  - `CloudWatchFullAccess` (optional)

**Create Access Key:**
1. User → Security credentials → Create access key
2. Use case: "Command Line Interface (CLI)"
3. **Save both keys securely:**
   - Access Key ID (starts with `AKIA...`)
   - Secret Access Key (shown only once)

### 2.3 Configure AWS CLI

```bash
# Configure for ubuntu user
aws configure
# AWS Access Key ID: [paste your key]
# AWS Secret Access Key: [paste your secret]
# Default region: ap-southeast-2
# Default output format: json

# Configure for root user (needed for backup script)
sudo aws configure
# Enter same credentials
```

**Verification:**
```bash
aws s3 ls
# Should show empty list or existing buckets (no errors)
```

### 2.4 Create S3 Bucket

```bash
# Create bucket (replace with your unique name if needed)
aws s3 mb s3://your-company-n8n-backups --region ap-southeast-2

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket your-company-n8n-backups \
  --versioning-configuration Status=Enabled

# Verify
aws s3 ls
```

**Expected output:** Your bucket listed

### 2.5 Create Backup Script

```bash
sudo nano /usr/local/bin/n8n-backup-enterprise.sh
```

**Paste this script:**

```bash
#!/bin/bash
set -e

# Configuration
BACKUP_DIR="/var/backups/n8n"
S3_BUCKET="s3://your-company-n8n-backups"
DATA_PATH="/var/lib/docker/volumes/n8n_data/_data"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d-%H%M%S)
HOSTNAME=$(hostname)

# Logging
LOG_FILE="/var/log/n8n-backup.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "===== n8n Backup Started: $(date) ====="

# Create local backup directory
mkdir -p $BACKUP_DIR

# Backup database with WAL checkpoint
sqlite3 $DATA_PATH/database.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
cp $DATA_PATH/database.sqlite $BACKUP_DIR/database-$DATE.sqlite

# Backup entire data directory
tar -czf $BACKUP_DIR/n8n-full-$DATE.tar.gz -C $DATA_PATH .

# Upload to S3
aws s3 cp $BACKUP_DIR/database-$DATE.sqlite \
  $S3_BUCKET/$HOSTNAME/database/ \
  --storage-class STANDARD_IA

aws s3 cp $BACKUP_DIR/n8n-full-$DATE.tar.gz \
  $S3_BUCKET/$HOSTNAME/full/ \
  --storage-class STANDARD_IA

# Clean up local backups older than 7 days
find $BACKUP_DIR -name "database-*.sqlite" -mtime +7 -delete
find $BACKUP_DIR -name "n8n-full-*.tar.gz" -mtime +7 -delete

# Clean up S3 backups older than retention period
aws s3 ls $S3_BUCKET/$HOSTNAME/database/ | while read -r line; do
  fileName=$(echo $line | awk '{print $4}')
  if [[ ! -z "$fileName" ]]; then
    fileDate=$(echo $fileName | grep -oP '\d{8}' || echo "")
    if [[ ! -z "$fileDate" ]]; then
      daysOld=$(( ($(date +%s) - $(date -d "$fileDate" +%s 2>/dev/null || echo 0)) / 86400 ))
      if [[ $daysOld -gt $RETENTION_DAYS ]]; then
        aws s3 rm $S3_BUCKET/$HOSTNAME/database/$fileName
      fi
    fi
  fi
done

echo "===== n8n Backup Completed: $(date) ====="
echo "Database backup: $S3_BUCKET/$HOSTNAME/database/database-$DATE.sqlite"
echo "Full backup: $S3_BUCKET/$HOSTNAME/full/n8n-full-$DATE.tar.gz"
```

**Save:** `Ctrl+X`, then `Y`, then `Enter`

**Make executable:**
```bash
sudo chmod +x /usr/local/bin/n8n-backup-enterprise.sh
```

### 2.6 Test Backup Script

```bash
sudo /usr/local/bin/n8n-backup-enterprise.sh
```

**Expected output:**
```
===== n8n Backup Started: Wed Dec 24 05:xx:xx UTC 2025 =====
0|0|0
upload: database-20251224-xxxxxx.sqlite to s3://...
upload: n8n-full-20251224-xxxxxx.tar.gz to s3://...
===== n8n Backup Completed: Wed Dec 24 05:xx:xx UTC 2025 =====
```

**Verify in S3:**
```bash
aws s3 ls s3://your-company-n8n-backups/your-hostname/database/
aws s3 ls s3://your-company-n8n-backups/your-hostname/full/
```

### 2.7 Set Up Automated Daily Backups

**Create systemd service:**
```bash
sudo nano /etc/systemd/system/n8n-backup.service
```

**Paste:**
```ini
[Unit]
Description=n8n Enterprise Backup
After=network.target docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/n8n-backup-enterprise.sh
User=root
StandardOutput=journal
StandardError=journal
```

**Create systemd timer:**
```bash
sudo nano /etc/systemd/system/n8n-backup.timer
```

**Paste:**
```ini
[Unit]
Description=n8n Backup Timer - Daily at 2 AM Singapore Time
Requires=n8n-backup.service

[Timer]
OnCalendar=18:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable n8n-backup.timer
sudo systemctl start n8n-backup.timer
```

**Verify timer is active:**
```bash
sudo systemctl status n8n-backup.timer
sudo systemctl list-timers --all | grep n8n
```

**Expected output:**
```
● n8n-backup.timer - n8n Backup Timer - Daily at 2 AM Singapore Time
     Loaded: loaded
     Active: active (waiting)
    Trigger: [Next run time shown]
```

---

## Verification and Monitoring

### Check Backup Status

```bash
# View backup logs
sudo tail -f /var/log/n8n-backup.log

# Check last 20 lines of log
sudo tail -n 20 /var/log/n8n-backup.log

# Check timer status
sudo systemctl status n8n-backup.timer

# List all timers
sudo systemctl list-timers --all
```

### Verify S3 Backups

```bash
# List database backups
aws s3 ls s3://your-company-n8n-backups/your-hostname/database/

# List full backups
aws s3 ls s3://your-company-n8n-backups/your-hostname/full/

# Check backup size
aws s3 ls s3://your-company-n8n-backups/your-hostname/ --recursive --human-readable
```

### Verify EBS Snapshots

**Navigate to:** EC2 → Snapshots

**Check for:**
- Snapshots tagged with your policy name
- Description includes "Created by Data Lifecycle Manager"
- Status: "Completed"

### Manual Backup Trigger

```bash
# Run backup manually for testing
sudo /usr/local/bin/n8n-backup-enterprise.sh

# Check if it completed successfully
echo $?
# Should output: 0
```

---

## Backup Restoration Procedures

### Restore from S3 Database Backup

```bash
# 1. List available backups
aws s3 ls s3://your-company-n8n-backups/your-hostname/database/

# 2. Download specific backup
aws s3 cp s3://your-company-n8n-backups/your-hostname/database/database-YYYYMMDD-HHMMSS.sqlite /tmp/restore.sqlite

# 3. Stop n8n
sudo docker stop n8n

# 4. Backup current database (safety)
sudo cp /var/lib/docker/volumes/n8n_data/_data/database.sqlite /tmp/current-backup.sqlite

# 5. Replace database
sudo cp /tmp/restore.sqlite /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# 6. Set correct permissions
sudo chown ubuntu:ubuntu /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# 7. Start n8n
sudo docker start n8n

# 8. Verify
sudo docker logs n8n
```

### Restore from S3 Full Backup

```bash
# 1. Download full backup
aws s3 cp s3://your-company-n8n-backups/your-hostname/full/n8n-full-YYYYMMDD-HHMMSS.tar.gz /tmp/

# 2. Stop n8n
sudo docker stop n8n

# 3. Backup current data
sudo tar -czf /tmp/current-n8n-data.tar.gz -C /var/lib/docker/volumes/n8n_data/_data .

# 4. Clear current data
sudo rm -rf /var/lib/docker/volumes/n8n_data/_data/*

# 5. Extract backup
sudo tar -xzf /tmp/n8n-full-YYYYMMDD-HHMMSS.tar.gz -C /var/lib/docker/volumes/n8n_data/_data/

# 6. Set permissions
sudo chown -R ubuntu:ubuntu /var/lib/docker/volumes/n8n_data/_data

# 7. Start n8n
sudo docker start n8n
```

### Restore from EBS Snapshot

**Via AWS Console:**
1. EC2 → Snapshots
2. Select snapshot → Actions → Create Volume
3. Attach new volume to instance
4. Mount and copy data
5. Or create new instance from snapshot

---

## Backup Schedule Summary

| Backup Type | Frequency | Time (Singapore) | Time (UTC) | Retention | Location |
|-------------|-----------|------------------|------------|-----------|----------|
| EBS Snapshots | Daily | 12:00 AM | 16:00 | 7 days | AWS Snapshots |
| S3 Database | Daily | 2:00 AM | 18:00 | 30 days | S3 Bucket |
| S3 Full Backup | Daily | 2:00 AM | 18:00 | 30 days | S3 Bucket |

---

## Troubleshooting

### Backup Script Fails

```bash
# Check logs
sudo tail -f /var/log/n8n-backup.log

# Test AWS credentials
aws s3 ls

# Test as root
sudo aws s3 ls

# Manually run script with verbose output
sudo bash -x /usr/local/bin/n8n-backup-enterprise.sh
```

### Timer Not Running

```bash
# Check timer status
sudo systemctl status n8n-backup.timer

# Check service status
sudo systemctl status n8n-backup.service

# View timer logs
sudo journalctl -u n8n-backup.timer -f

# Restart timer
sudo systemctl restart n8n-backup.timer
```

### S3 Upload Issues

```bash
# Verify AWS credentials
aws sts get-caller-identity

# Check S3 bucket permissions
aws s3api get-bucket-acl --bucket your-company-n8n-backups

# Test upload manually
echo "test" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://your-company-n8n-backups/test.txt
```

### EBS Snapshot Issues

**Check DLM policy:**
- EC2 → Lifecycle Manager → Select policy
- Verify "State" is "Enabled"
- Check "Execution role" has proper permissions
- Review policy details match configuration

---

## Security Best Practices

### IAM User Permissions
- ✅ Use dedicated IAM user for backups
- ✅ Minimum required permissions (S3FullAccess)
- ✅ Rotate access keys every 90 days
- ✅ Enable MFA on IAM user (optional but recommended)

### S3 Bucket Security
- ✅ Versioning enabled
- ✅ Private bucket (no public access)
- ✅ Encryption at rest (AWS managed)
- ✅ Lifecycle policies for cost optimization

### Access Control
- ✅ SSH key authentication only
- ✅ Security groups properly configured
- ✅ AWS credentials stored securely
- ✅ Backup logs protected (chmod 600)

---

## Cost Optimization

### EBS Snapshots
- Incremental snapshots (only changed blocks)
- 7-day retention keeps costs low
- Estimated: ~$0.05/GB/month

### S3 Storage
- STANDARD_IA (Infrequent Access) class
- 30-day retention
- Estimated: ~$0.0125/GB/month

### Total Estimated Monthly Cost
- EBS Snapshots: ~$0.40 (8GB × 7 days)
- S3 Storage: ~$0.10 (8GB × 30 days)
- **Total: ~$0.50/month**

---

## Maintenance Tasks

### Monthly
- [ ] Review backup logs for failures
- [ ] Verify at least one backup is restorable
- [ ] Check S3 bucket size and costs
- [ ] Review EBS snapshot count

### Quarterly
- [ ] Test full restoration procedure
- [ ] Review and update retention policies
- [ ] Rotate IAM access keys
- [ ] Update backup documentation

### Annually
- [ ] Disaster recovery drill
- [ ] Review backup strategy
- [ ] Update all scripts and tools
- [ ] Audit security configurations

---

## Quick Reference Commands

```bash
# Manual backup
sudo /usr/local/bin/n8n-backup-enterprise.sh

# View logs
sudo tail -f /var/log/n8n-backup.log

# Check timer
sudo systemctl status n8n-backup.timer

# List S3 backups
aws s3 ls s3://your-company-n8n-backups/your-hostname/database/

# Check EBS snapshots
# Via AWS Console: EC2 → Snapshots

# Stop timer
sudo systemctl stop n8n-backup.timer

# Start timer
sudo systemctl start n8n-backup.timer

# Restart timer
sudo systemctl restart n8n-backup.timer
```

---

## Support and Resources

- **AWS Documentation:** https://docs.aws.amazon.com/
- **n8n Documentation:** https://docs.n8n.io/
- **S3 Pricing:** https://aws.amazon.com/s3/pricing/
- **EBS Pricing:** https://aws.amazon.com/ebs/pricing/

---

## Document Version

- **Created:** December 24, 2025
- **Last Updated:** December 24, 2025
- **Author:** Your Name
- **System:** n8n on AWS EC2 (ap-southeast-2)

---

## Notes

- All times are in UTC unless specified
- Singapore time = UTC+8
- Backup script includes automatic cleanup
- S3 versioning provides additional protection
- EBS snapshots are incremental and cost-effective
- Always test restoration before relying on backups
