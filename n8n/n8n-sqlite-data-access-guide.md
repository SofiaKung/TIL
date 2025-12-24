# n8n Data Access and SQLite Query Guide

## Overview
This guide covers how to access, inspect, and query n8n workflow data stored in the SQLite database on your AWS EC2 instance.

---

## System Information
- **n8n Instance:** EC2 i-XXXXXXXXXXXXX
- **Data Storage:** Docker volume `n8n_data`
- **Database Location:** `/var/lib/docker/volumes/n8n_data/_data/database.sqlite`
- **Database Type:** SQLite 3
- **Access Method:** SSH to EC2, then use sqlite3 CLI

---

## n8n Data Storage Architecture

### Docker Volume Structure

```
n8n_data (Docker volume)
└── /var/lib/docker/volumes/n8n_data/_data/
    ├── database.sqlite          # Main workflow database
    ├── database.sqlite-shm      # Shared memory file
    ├── database.sqlite-wal      # Write-Ahead Log
    ├── config                   # n8n configuration
    ├── binaryData/              # Uploaded files
    ├── git/                     # Git integration data
    ├── nodes/                   # Custom nodes
    ├── ssh/                     # SSH keys
    └── n8nEventLog.log         # Application logs
```

### Database Files Explained

| File | Size | Purpose |
|------|------|---------|
| `database.sqlite` | ~572KB | Main database with workflows, credentials, executions |
| `database.sqlite-wal` | ~4MB | Write-Ahead Log (pending writes) |
| `database.sqlite-shm` | ~32KB | Shared memory for WAL mode |

**Total Database Size:** ~4.6MB

---

## Accessing n8n Data

### Method 1: Via SQLite CLI (Direct Access)

#### Step 1: SSH into EC2 Instance

```bash
# From your local machine
ssh -i "your-key-pair.pem" ubuntu@your-ec2-instance.ap-southeast-2.compute.amazonaws.com
```

#### Step 2: Install SQLite3 (if not installed)

```bash
sudo apt-get update
sudo apt-get install -y sqlite3
```

#### Step 3: Access the Database

```bash
# Navigate to data directory (requires sudo)
sudo -i
cd /var/lib/docker/volumes/n8n_data/_data

# Open database
sqlite3 database.sqlite
```

**Alternative (direct command):**
```bash
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite
```

### Method 2: Via Docker Container

```bash
# Access SQLite from inside the running n8n container
sudo docker exec -it n8n sqlite3 /home/node/.n8n/database.sqlite
```

---

## SQLite Basic Commands

### Inside SQLite Prompt

```sql
-- Show all tables in the database
.tables

-- Show table structure/schema
.schema workflow_entity
.schema credentials_entity
.schema execution_entity

-- Set output formatting
.mode column
.headers on

-- Show current database
.databases

-- Get help
.help

-- Exit SQLite
.quit
```

---

## Querying n8n Workflows

### List All Workflows

```sql
-- Basic workflow list
SELECT id, name, active FROM workflow_entity;
```

**Example Output:**
```
id                | name                                          | active
abc123def456gh78 | LinkedIn Content Generation System - Complete | 0
xyz789uvw012ij34 | Email Automation Workflow                     | 1
```

### Get Workflow Details

```sql
-- Get specific workflow by ID
SELECT 
    id,
    name,
    active,
    createdAt,
    updatedAt
FROM workflow_entity 
WHERE id = 'abc123def456gh78';
```

### View Workflow Configuration

```sql
-- Get workflow nodes and connections
SELECT 
    id,
    name,
    nodes,
    connections
FROM workflow_entity 
WHERE id = 'abc123def456gh78';
```

**Note:** `nodes` and `connections` contain JSON data

### Count Workflows

```sql
-- Total workflows
SELECT COUNT(*) as total_workflows FROM workflow_entity;

-- Active workflows
SELECT COUNT(*) as active_workflows 
FROM workflow_entity 
WHERE active = 1;

-- Inactive workflows
SELECT COUNT(*) as inactive_workflows 
FROM workflow_entity 
WHERE active = 0;
```

### Search Workflows by Name

```sql
-- Find workflows containing "LinkedIn"
SELECT id, name, active 
FROM workflow_entity 
WHERE name LIKE '%LinkedIn%';

-- Case-insensitive search
SELECT id, name, active 
FROM workflow_entity 
WHERE LOWER(name) LIKE LOWER('%content%');
```

---

## Querying Workflow Executions

### Recent Executions

```sql
-- List recent executions
SELECT 
    id,
    workflowId,
    mode,
    status,
    startedAt,
    stoppedAt
FROM execution_entity 
ORDER BY startedAt DESC 
LIMIT 10;
```

### Execution Statistics

```sql
-- Count executions by status
SELECT 
    status,
    COUNT(*) as count
FROM execution_entity 
GROUP BY status;

-- Executions per workflow
SELECT 
    w.name as workflow_name,
    COUNT(e.id) as execution_count
FROM workflow_entity w
LEFT JOIN execution_entity e ON w.id = e.workflowId
GROUP BY w.id, w.name
ORDER BY execution_count DESC;
```

---

## Querying Credentials

### List All Credentials

```sql
-- Basic credential list (without sensitive data)
SELECT 
    id,
    name,
    type,
    createdAt
FROM credentials_entity;
```

**⚠️ Security Note:** Credentials are encrypted. The `data` field contains encrypted values.

---

## Advanced Queries

### Workflow with Most Nodes

```sql
-- Find complex workflows
SELECT 
    id,
    name,
    LENGTH(nodes) as nodes_size,
    active
FROM workflow_entity 
ORDER BY nodes_size DESC 
LIMIT 5;
```

### Recently Updated Workflows

```sql
-- Last modified workflows
SELECT 
    id,
    name,
    updatedAt,
    active
FROM workflow_entity 
ORDER BY updatedAt DESC 
LIMIT 10;
```

### Workflows Created in Date Range

```sql
-- Workflows created this month
SELECT 
    id,
    name,
    createdAt
FROM workflow_entity 
WHERE createdAt >= datetime('now', 'start of month');
```

---

## Exporting Data

### Export Query Results to CSV

```sql
-- Inside sqlite3
.mode csv
.output /tmp/workflows.csv
SELECT id, name, active, createdAt FROM workflow_entity;
.output stdout
.mode column
```

### Export Entire Table

```bash
# From command line (not in sqlite3)
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite \
  "SELECT * FROM workflow_entity;" \
  -csv > ~/workflows_export.csv
```

### Export Workflow JSON

```sql
-- Export specific workflow
.output /tmp/workflow_udlfOvJ8ksD4o8Mt.json
SELECT json_object(
    'id', id,
    'name', name,
    'nodes', nodes,
    'connections', connections,
    'active', active
) 
FROM workflow_entity 
WHERE id = 'abc123def456gh78';
.output stdout
```

---

## Database Backup via SQLite

### Create Database Backup

```bash
# Backup with WAL checkpoint
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"

# Copy database file
sudo cp /var/lib/docker/volumes/n8n_data/_data/database.sqlite \
  ~/n8n-database-backup-$(date +%Y%m%d).sqlite
```

### Verify Backup Integrity

```bash
# Check database integrity
sudo sqlite3 ~/n8n-database-backup-$(date +%Y%m%d).sqlite "PRAGMA integrity_check;"
```

**Expected output:** `ok`

---

## Common Database Tables

### Main Tables Overview

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `workflow_entity` | Workflow definitions | id, name, nodes, connections, active |
| `execution_entity` | Workflow execution history | id, workflowId, status, startedAt, data |
| `credentials_entity` | Stored credentials | id, name, type, data (encrypted) |
| `tag_entity` | Workflow tags | id, name |
| `workflow_tag_mapping` | Workflow-tag relationships | workflowId, tagId |
| `settings` | n8n settings | key, value |
| `variables` | Environment variables | id, key, value |

### View All Tables

```sql
-- List all tables
.tables

-- Get row count for each table
SELECT 'workflow_entity' as table_name, COUNT(*) as rows FROM workflow_entity
UNION ALL
SELECT 'execution_entity', COUNT(*) FROM execution_entity
UNION ALL
SELECT 'credentials_entity', COUNT(*) FROM credentials_entity;
```

---

## Troubleshooting

### Database Locked Error

```bash
# If you get "database is locked"
# 1. Stop n8n temporarily
sudo docker stop n8n

# 2. Run your query
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# 3. Restart n8n
sudo docker start n8n
```

### WAL Mode Issues

```sql
-- Check WAL mode
PRAGMA journal_mode;

-- Checkpoint WAL file
PRAGMA wal_checkpoint(TRUNCATE);

-- Disable WAL (not recommended while n8n is running)
-- PRAGMA journal_mode=DELETE;
```

### Permission Denied

```bash
# Use sudo for all database access
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# Or switch to root
sudo -i
```

---

## Data Validation Queries

### Check for Orphaned Executions

```sql
-- Executions without workflows
SELECT e.id, e.workflowId 
FROM execution_entity e
LEFT JOIN workflow_entity w ON e.workflowId = w.id
WHERE w.id IS NULL;
```

### Check Workflow Consistency

```sql
-- Workflows without executions
SELECT w.id, w.name, COUNT(e.id) as execution_count
FROM workflow_entity w
LEFT JOIN execution_entity e ON w.id = e.workflowId
GROUP BY w.id, w.name
HAVING execution_count = 0;
```

### Database Statistics

```sql
-- Database size info
SELECT page_count * page_size as size 
FROM pragma_page_count(), pragma_page_size();

-- Table sizes
SELECT 
    name as table_name,
    SUM(pgsize) as size_bytes
FROM dbstat
GROUP BY name
ORDER BY size_bytes DESC;
```

---

## Best Practices

### ✅ DO:
- Always use `sudo` when accessing database files
- Backup database before making direct changes
- Use read-only queries when possible
- Checkpoint WAL before backups: `PRAGMA wal_checkpoint(TRUNCATE);`
- Test queries on a backup copy first

### ❌ DON'T:
- Don't modify database while n8n is running
- Don't delete records without backups
- Don't decrypt credentials manually
- Don't share database files (contains encrypted credentials)
- Don't run long queries during peak usage

---

## Security Considerations

### Sensitive Data

The database contains:
- ✅ **Encrypted:** Credentials, API keys, passwords
- ⚠️ **Unencrypted:** Workflow logic, node configurations, execution data

### Access Control

```bash
# Check database file permissions
ls -lah /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# Should be owned by ubuntu:ubuntu
# Permissions: -rw-r--r--
```

### Credential Safety

```sql
-- ⚠️ NEVER run this in production
-- SELECT data FROM credentials_entity;

-- Instead, manage credentials via n8n UI
-- Credentials are encrypted at rest
```

---

## Quick Reference Commands

### Access Database

```bash
# SSH to EC2
ssh -i "n8n-n8n-tech-key-pair.pem" ubuntu@your-instance.com

# Direct access
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite

# Via container
sudo docker exec -it n8n sqlite3 /home/node/.n8n/database.sqlite
```

### Common Queries

```sql
-- List workflows
SELECT id, name, active FROM workflow_entity;

-- Count workflows
SELECT COUNT(*) FROM workflow_entity;

-- Recent executions
SELECT * FROM execution_entity ORDER BY startedAt DESC LIMIT 10;

-- Exit
.quit
```

### Backup Commands

```bash
# Checkpoint WAL
sudo sqlite3 /var/lib/docker/volumes/n8n_data/_data/database.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"

# Backup database
sudo cp /var/lib/docker/volumes/n8n_data/_data/database.sqlite ~/backup.sqlite

# Verify backup
sudo sqlite3 ~/backup.sqlite "PRAGMA integrity_check;"
```

---

## Example: Full Workflow Analysis

```sql
-- Complete workflow analysis query
SELECT 
    w.id,
    w.name,
    w.active,
    w.createdAt,
    w.updatedAt,
    COUNT(e.id) as total_executions,
    SUM(CASE WHEN e.status = 'success' THEN 1 ELSE 0 END) as successful,
    SUM(CASE WHEN e.status = 'error' THEN 1 ELSE 0 END) as failed,
    MAX(e.startedAt) as last_execution
FROM workflow_entity w
LEFT JOIN execution_entity e ON w.id = e.workflowId
GROUP BY w.id, w.name, w.active, w.createdAt, w.updatedAt
ORDER BY last_execution DESC;
```

---

## Resources

- **SQLite Documentation:** https://www.sqlite.org/docs.html
- **n8n Database Schema:** Check n8n source code for latest schema
- **SQLite Browser (GUI):** https://sqlitebrowser.org/ (for local analysis)

---

## Document Information

- **Created:** December 24, 2025
- **Last Updated:** December 24, 2025
- **Author:** Your Name
- **System:** n8n on AWS EC2
- **Database Version:** SQLite 3.45.1
- **n8n Data Location:** /var/lib/docker/volumes/n8n_data/_data

---

## Notes

- Database is in WAL (Write-Ahead Logging) mode for better performance
- All workflows shown in examples are from actual system
- Credentials table contains encrypted data - never export raw
- Always backup before making any direct database modifications
- For production changes, use n8n API or UI instead of direct SQL
