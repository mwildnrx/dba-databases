# MySQL CDC User Setup – Qlik Replicate - Debezium

> **Purpose:** This configuration creates a dedicated MySQL user for Change Data Capture (CDC), including the required permissions for reading data, accessing replication metadata, and recovering XA transactions.

## Create CDC User

Create the `cdc` user and configure authentication using `caching_sha2_password`.

```sql
CREATE USER 'cdc'@'%' IDENTIFIED WITH caching_sha2_password BY 'mysqldb';
```

> **Note:** Replace the example password with a secure password before using this configuration in production.

---

## Grant Required Privileges

### XA Transaction Recovery

Grant permission to recover prepared XA transactions.

```sql
GRANT XA_RECOVER_ADMIN ON *.* TO 'cdc'@'%';
```

### Source Database Access

Grant read-only access to the `cdc` database.

```sql
GRANT SELECT ON `cdc`.* TO 'cdc'@'%';
```

### Replication Access

Grant permissions required to read replication and binary log metadata.

```sql
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'cdc'@'%';
```

---

## Apply Privilege Changes

```sql
FLUSH PRIVILEGES;
```

---

# Complete SQL Configuration

```sql
-- Create CDC user
CREATE USER 'cdc'@'%' IDENTIFIED WITH caching_sha2_password BY 'mysqldb';

-- Allow XA transaction recovery
GRANT XA_RECOVER_ADMIN ON *.* TO 'cdc'@'%';

-- Grant read access to the source database
GRANT SELECT ON `cdc`.* TO 'cdc'@'%';

-- Grant replication-related privileges
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'cdc'@'%';

-- Apply privilege changes
FLUSH PRIVILEGES;
```

---

# Verify User Configuration

Check the privileges assigned to the CDC user:

```sql
SHOW GRANTS FOR 'cdc'@'%';
```

Expected privileges include:

* `XA_RECOVER_ADMIN`
* `SELECT`
* `REPLICATION SLAVE`
* `REPLICATION CLIENT`

---

# Security Recommendation

For production environments, consider restricting the user to a specific host instead of using `%`.

Example:

```sql
CREATE USER 'cdc'@'10.10.10.%'
IDENTIFIED WITH caching_sha2_password
BY '<secure_password>';
```

This limits database access to clients within the specified network range.
