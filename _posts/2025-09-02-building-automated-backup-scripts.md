---
title: "Building Automated Backup Scripts for My Azure-to-Local Workflow"
date: 2025-09-02 02:32:00 +0400
categories: [DevOps, Scripting, Backups]
tags: [bash, linux, windows, azure, backup, cron, automation, scp, databases]
author: osama
image:
  path: /assets/img/posts/ubuntu-cloud-backup-local.jpg
  alt: Linux Automated Backup Scripts
description: Bash scripts for automated backups of NGINX, SQL Server, PostgreSQL, MongoDB, and other services from Linux hosts to a remote server over SSH/SCP.
pin: false
comments: true

---

In our environment, we already had **Azure cloud backups enabled** for business continuity. But the team wanted **nightly local copies** as an extra safety net. The requirement was simple: every night, the latest configs and data should also land on our Windows server in the office.  

I decided to build a lightweight, script-driven solution for this. No extra licensing, no heavy agents — just plain Bash and `scp`.

---

## What the scripts do

The scripts handle:

- Snapshot of **NGINX configs**
- Stop/backup/restart of **SQL Server** Docker volumes
- **PostgreSQL** logical dump (`pg_dumpall`)
- **MongoDB** dumps from containers
- Packaging of **Core-Fabric** and **Mongo dev stacks**

Each archive is timestamped and shipped via `scp` to `D:/Backups/...` on the Windows host.

These jobs run **every night via cron**, so by morning, both Azure and local copies are available.

---

## Why build custom scripts?

- **Extra layer**: Azure cloud backups were already there, but local backups offered faster restore for dev/test.  
- **Transparency**: Archives are just `.tar.gz` or `.sql.gz`.  
- **Flexibility**: Can stop containers where consistency matters.  
- **Automation**: Runs unattended with logs.  
- **Cross-platform**: Linux sources, Windows destination.

---

## Repository structure

Full repo: [linux-backup-scripts](https://github.com/maxdorx/linux-backup-scripts)

```
linux-backup-scripts/
├─ scripts/
│  ├─ nginx-backup.sh
│  ├─ sqlserver-backup.sh
│  ├─ postgres-backup.sh
│  ├─ mongo-backup.sh
│  ├─ core-fabric-backup.sh
│  └─ mongo-dev-backup.sh
├─ sample.env
├─ README.md
├─ LICENSE
└─ .gitignore
```

---

## Example: NGINX config snapshot

```bash
scp -P $REMOTE_PORT $tmp/${stamp}-nginx.conf${REMOTE_USER}@${REMOTE_HOST}:$DEST_DIR
```

This takes `/etc/nginx/nginx.conf`, stamps it with the date, and sends it to the Windows path.

---

## Database backups in brief

- **SQL Server** → down Docker → tar → transfer → up Docker  
- **PostgreSQL** → `pg_dumpall` → gzip → transfer  
- **MongoDB** → `mongodump` inside container → copy out → tar → transfer  

Each writes to its own log under `/var/backups`.

---

## Scheduling with cron

```bash
# Postgres backup nightly at 2:00
0 2 * * * cd /opt/linux-backup-scripts   && env $(cat .env | xargs) scripts/postgres-backup.sh   >>/var/log/postgres-backup.cron 2>&1
```

This is how they ran **every night without fail**.

---

## Restores

- **Postgres**  
  ```bash
  gunzip -c postgres-YYYYMMDD.sql.gz | psql -U postgres
  ```

- **MongoDB**  
  ```bash
  tar -xzf mongodb_*.tar.gz
  mongorestore dump/
  ```

- **File archives**  
  ```bash
  tar -xzf backup.tar.gz -C /restore/path
  ```

---

## Security notes

- `.env` is ignored in Git.  
- Remote user `svc-backup` is least-privileged.  
- Key authentication over password.  
- Logs rotated and monitored.  

---

## Improvements on the roadmap

- Replace `scp` with `rsync` for speed and delta sync.  
- Add SHA256 checksums.  
- Encrypt backups at rest.  
- Wire failure notifications to Slack/email.  

---

## Closing

I used these scripts on our Azure Linux servers and **send backups to our on-prem Windows server** every night via SSH/SCP.

---

⚠️ **Note**: All IPs, usernames, ports, and paths shown here are placeholders. Replace them with values from your own environment before running the scripts.