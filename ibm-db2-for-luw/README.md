# DB2 HADR Cheat Sheet – Primary & Standby Setup

> **Purpose:** This document provides a quick reference for configuring IBM Db2 High Availability Disaster Recovery (HADR) between a Primary and Standby database server.

## Architecture Overview

```text
+---------------------------+       HADR Replication       +---------------------------+
|       PRIMARY NODE        | <-------------------------> |       STANDBY NODE        |
|                           |                             |                           |
| Database: <database>      |                             | Database: <database>      |
| HADR Local Service:       |                             | HADR Local Service:       |
| DB2_HADR_1                |                             | DB2_HADR_2                |
| Port: 55001               |                             | Port: 55002               |
+---------------------------+                             +---------------------------+
          Primary                                                    Standby
```

---

# 1. Prerequisites

Before configuring HADR, ensure the following requirements are met:

* Db2 is installed on both Primary and Standby nodes.
* The Db2 instance name is configured correctly.
* Network connectivity between both servers is available.
* HADR ports are reachable between the Primary and Standby nodes.
* The database backup from the Primary node is available on the Standby node.
* The database names are consistent between both nodes.

Example placeholders used in this document:

| Placeholder      | Description             |
| ---------------- | ----------------------- |
| `<database>`     | Db2 database name       |
| `<host_primary>` | Primary server hostname |
| `<host_standby>` | Standby server hostname |
| `<timestamp>`    | Backup timestamp        |

---

# 2. Primary Node – Create Database

Create the database with UTF-8 encoding and Indonesian territory settings.

```bash
db2 CREATE DATABASE <database> USING CODESET UTF-8 TERRITORY ID
```

Verify the database:

```bash
db2 list db directory
```

---

# 3. Primary Node – Configure Log Archival

Configure disk-based log archival, which is required for HADR recovery.

```bash
db2 UPDATE DB CFG FOR <database> \
USING LOGARCHMETH1 DISK:/data/db2-logs/
```

Verify the configuration:

```bash
db2 get db cfg for <database> | grep LOGARCHMETH1
```

---

# 4. Primary Node – Create Initial Offline Backup

Create a full offline database backup.

```bash
db2 BACKUP DATABASE <database> TO /data/db2-backup
```

This backup can be retained as an initial recovery point.

---

# 5. Primary Node – Optional Schema and Table Test

Connect to the database:

```bash
db2 connect to <database>
```

Create a test schema:

```bash
db2 "CREATE SCHEMA CDC"
```

Set the active schema:

```bash
db2 "SET CURRENT SCHEMA CDC"
```

Create a test table:

```bash
db2 "CREATE TABLE TEST (
    ID INTEGER NOT NULL
        GENERATED ALWAYS AS IDENTITY
        (START WITH 1 INCREMENT BY 1),
    NAME VARCHAR(100),
    PRIMARY KEY (ID)
)"
```

Verify the table:

```bash
db2 "LIST TABLES FOR SCHEMA CDC"
```

---

# 6. Primary Node – Create Online Backup for Standby

Create an online backup including the required transaction logs.

```bash
db2 BACKUP DATABASE <database> ONLINE \
TO /data/db2-backup \
INCLUDE LOGS
```

List the backup files:

```bash
ls -lah /data/db2-backup
```

---

# 7. Copy Backup to Standby Node

Copy the database backup from the Primary server to the Standby server.

```bash
scp -rp /data/db2-backup/* \
db2inst1@<host_standby>:/data/db2-backup/
```

Verify the backup files on the Standby node:

```bash
ls -lah /data/db2-backup
```

---

# 8. Standby Node – Restore Database

Restore the database using the backup created on the Primary node.

```bash
db2 RESTORE DATABASE <database> \
FROM "/data/db2-backup" \
TAKEN AT <timestamp> \
REPLACE HISTORY FILE
```

> Ensure that `<timestamp>` matches the timestamp of the backup image being restored.

Verify the database state:

```bash
db2 list active databases
```

---

# 9. Configure Db2 Service Port

Perform this step on **both Primary and Standby nodes**.

Check the Db2 service configuration:

```bash
db2 get dbm cfg | grep SVCENAME
```

---

# 10. Register HADR Services

Perform this step on **both Primary and Standby nodes**.

Add the HADR service definitions to `/etc/services`.

```bash
echo "DB2_HADR_1 55001/tcp" | sudo tee -a /etc/services

echo "DB2_HADR_2 55002/tcp" | sudo tee -a /etc/services
```

Verify the configuration:

```bash
grep "DB2_HADR_" /etc/services
```

Expected configuration:

```text
DB2_HADR_1 55001/tcp
DB2_HADR_2 55002/tcp
```

---

# 11. Primary Node – Configure Required Logging

Configure the required database logging and recovery settings.

```bash
db2 UPDATE DB CFG FOR <database> USING LOGRETAIN ON

db2 UPDATE DB CFG FOR <database> USING TRACKMOD ON

db2 UPDATE DB CFG FOR <database> USING LOGINDEXBUILD ON

db2 UPDATE DB CFG FOR <database> USING INDEXREC RESTART
```

---

# 12. Primary Node – Configure Alternate Server

Configure the Standby server as the alternate server for client rerouting.

```bash
db2 UPDATE ALTERNATE SERVER FOR DATABASE <database> \
USING HOSTNAME <host_standby> \
PORT 60000
```

---

# 13. Primary Node – Configure HADR

Configure the HADR parameters on the Primary node.

```bash
db2 UPDATE DB CFG FOR <database> \
USING HADR_LOCAL_HOST <host_primary>

db2 UPDATE DB CFG FOR <database> \
USING HADR_LOCAL_SVC DB2_HADR_1

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_HOST <host_standby>

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_SVC DB2_HADR_2

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_INST db2inst1

db2 UPDATE DB CFG FOR <database> \
USING HADR_SYNCMODE ASYNC

db2 UPDATE DB CFG FOR <database> \
USING HADR_TIMEOUT 3

db2 UPDATE DB CFG FOR <database> \
USING HADR_PEER_WINDOW 120
```

### Primary HADR Summary

| Parameter          | Value            |
| ------------------ | ---------------- |
| `HADR_LOCAL_HOST`  | `<host_primary>` |
| `HADR_LOCAL_SVC`   | `DB2_HADR_1`     |
| `HADR_REMOTE_HOST` | `<host_standby>` |
| `HADR_REMOTE_SVC`  | `DB2_HADR_2`     |
| `HADR_REMOTE_INST` | `db2inst1`       |
| `HADR_SYNCMODE`    | `ASYNC`          |
| `HADR_TIMEOUT`     | `3`              |
| `HADR_PEER_WINDOW` | `120`            |

---

# 14. Standby Node – Configure Required Logging

Configure the required database logging and recovery settings.

```bash
db2 UPDATE DB CFG FOR <database> USING LOGRETAIN ON

db2 UPDATE DB CFG FOR <database> USING TRACKMOD ON

db2 UPDATE DB CFG FOR <database> USING LOGINDEXBUILD ON

db2 UPDATE DB CFG FOR <database> USING INDEXREC RESTART
```

---

# 15. Standby Node – Configure Alternate Server

Configure the Primary server as the alternate server.

```bash
db2 UPDATE ALTERNATE SERVER FOR DATABASE <database> \
USING HOSTNAME <host_primary> \
PORT 60000
```

---

# 16. Standby Node – Configure HADR

Configure the HADR parameters on the Standby node.

```bash
db2 UPDATE DB CFG FOR <database> \
USING HADR_LOCAL_HOST <host_standby>

db2 UPDATE DB CFG FOR <database> \
USING HADR_LOCAL_SVC DB2_HADR_2

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_HOST <host_primary>

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_SVC DB2_HADR_1

db2 UPDATE DB CFG FOR <database> \
USING HADR_REMOTE_INST db2inst1

db2 UPDATE DB CFG FOR <database> \
USING HADR_SYNCMODE ASYNC

db2 UPDATE DB CFG FOR <database> \
USING HADR_TIMEOUT 3

db2 UPDATE DB CFG FOR <database> \
USING HADR_PEER_WINDOW 120
```

### Standby HADR Summary

| Parameter          | Value            |
| ------------------ | ---------------- |
| `HADR_LOCAL_HOST`  | `<host_standby>` |
| `HADR_LOCAL_SVC`   | `DB2_HADR_2`     |
| `HADR_REMOTE_HOST` | `<host_primary>` |
| `HADR_REMOTE_SVC`  | `DB2_HADR_1`     |
| `HADR_REMOTE_INST` | `db2inst1`       |
| `HADR_SYNCMODE`    | `ASYNC`          |
| `HADR_TIMEOUT`     | `3`              |
| `HADR_PEER_WINDOW` | `120`            |

---

# 17. Verify HADR Configuration

Verify the HADR parameters on both servers.

```bash
db2 get db cfg for <database> | grep HADR
```

Verify the registered services:

```bash
grep "DB2_HADR_" /etc/services
```

---

# 18. Start HADR

> **Important:** The startup order is mandatory. Always start the **Standby database first**, followed by the **Primary database**.

## Step 1 – Deactivate Database

Run on both Primary and Standby nodes:

```bash
db2 DEACTIVATE DATABASE <database>
```

---

## Step 2 – Start HADR on Standby

Run on the Standby node:

```bash
db2 START HADR ON DATABASE <database> AS STANDBY
```

---

## Step 3 – Start HADR on Primary

Run on the Primary node:

```bash
db2 START HADR ON DATABASE <database> AS PRIMARY
```

---

# 19. Verify HADR Status

Check the current HADR replication status:

```bash
db2pd -d <database> -hadr
```

The expected roles are:

```text
Primary Node
-----------
Role: PRIMARY

Standby Node
------------
Role: STANDBY
```

Check the database configuration:

```bash
db2 get db cfg for <database> | grep HADR
```

---

# 20. Manual HADR Takeover

Perform a manual takeover from the Standby server to promote it to Primary.

```bash
db2 TAKEOVER HADR ON DATABASE <database>
```

If authentication parameters are required:

```bash
db2 TAKEOVER HADR ON DATABASE <database> \
USER db2inst1 \
USING <password>
```

> After a successful takeover, the former Standby becomes the new Primary.

Verify the new role:

```bash
db2pd -d <database> -hadr
```

---

# 21. Stop HADR

Stop HADR from the node currently acting as Primary.

```bash
db2 STOP HADR ON DATABASE <database>
```

Verify the status:

```bash
db2pd -d <database> -hadr
```

---

# 22. Quick Command Reference

## Check HADR Status

```bash
db2pd -d <database> -hadr
```

## Check HADR Configuration

```bash
db2 get db cfg for <database> | grep HADR
```

## Start Standby

```bash
db2 START HADR ON DATABASE <database> AS STANDBY
```

## Start Primary

```bash
db2 START HADR ON DATABASE <database> AS PRIMARY
```

## Perform Takeover

```bash
db2 TAKEOVER HADR ON DATABASE <database>
```

## Stop HADR

```bash
db2 STOP HADR ON DATABASE <database>
```

---

# 23. Startup and Failover Sequence

```text
INITIAL HADR STARTUP
====================

Primary                         Standby
   |                                |
   |                                |
   |                          1. Start HADR
   |                             AS STANDBY
   |                                |
   |                                |
2. Start HADR                       |
   AS PRIMARY                       |
   |                                |
   +----------- HADR SYNC --------->|
   |<---------- HADR SYNC ----------|
   |                                |
```

```text
MANUAL TAKEOVER
===============

Before Takeover:

PRIMARY NODE                     STANDBY NODE
     PRIMARY        <----->        STANDBY

After Takeover:

FORMER PRIMARY                  FORMER STANDBY
     STANDBY       <----->        PRIMARY
```

---

# 24. Important Operational Notes

* Always verify that the Standby database has started successfully before starting HADR on the Primary.
* Ensure that HADR ports are reachable between both servers.
* Confirm the HADR synchronization state using `db2pd`.
* Perform regular backup validation and recovery testing.
* The `ASYNC` synchronization mode prioritizes application performance but may allow data loss during an unplanned failure.
* Review the appropriate HADR synchronization mode based on Recovery Point Objective (RPO) and Recovery Time Objective (RTO) requirements.
* Test client rerouting before implementing the configuration in a production environment.
* Avoid stopping HADR directly on the Standby unless the operational procedure specifically requires it.

---

# 25. Deployment Checklist

* [ ] Database created successfully on the Primary node.
* [ ] Log archival is configured.
* [ ] Initial database backup completed successfully.
* [ ] Online backup with logs completed successfully.
* [ ] Backup files copied to the Standby node.
* [ ] Database restored successfully on the Standby node.
* [ ] Db2 service ports verified.
* [ ] HADR services added to `/etc/services`.
* [ ] HADR ports are reachable between both nodes.
* [ ] Required logging parameters configured.
* [ ] Alternate server configured.
* [ ] HADR parameters configured on the Primary node.
* [ ] HADR parameters configured on the Standby node.
* [ ] Database deactivated on both nodes.
* [ ] HADR started on the Standby node.
* [ ] HADR started on the Primary node.
* [ ] HADR replication status verified.
* [ ] Primary role verified.
* [ ] Standby role verified.
* [ ] HADR synchronization tested.
