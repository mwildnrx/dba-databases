# Apache Hadoop, Hive, and PostgreSQL Metastore Setup

## Overview

This document contains the configuration and deployment notes for the following components:

* Apache Hadoop HDFS
* YARN
* MapReduce
* Apache Hive 3.1.3
* PostgreSQL Hive Metastore
* Hive Metastore Service
* HiveServer2
* HDFS Data Lake directory structure

## Architecture Overview

```text
                    +-------------------+
                    |   Hive / Beeline  |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |    HiveServer2    |
                    |  Port: 9852       |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |  Hive Metastore   |
                    |  Port: 9083       |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |    PostgreSQL     |
                    |  hive_metastore   |
                    +-------------------+

                              |
                              v

                    +-------------------+
                    |       HDFS        |
                    | data-lake storage |
                    +-------------------+
```

---

# 1. Hadoop Configuration

## `core-site.xml`

Configure the default HDFS filesystem.

```xml
<configuration>

  <property>
    <name>fs.default.name</name>
    <value>hdfs://data-warehouse.sibernetik.co.id:8020</value>
  </property>

</configuration>
```

---

## `hdfs-site.xml`

Configure HDFS replication and local storage directories for NameNode and DataNode.

```xml
<configuration>

  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>

  <property>
    <name>dfs.name.dir</name>
    <value>file:///data/apache-archive/hadoop_store/hdfs/namenode</value>
  </property>

  <property>
    <name>dfs.data.dir</name>
    <value>file:///data/apache-archive/hadoop_store/hdfs/datanode</value>
  </property>

</configuration>
```

---

## `mapred-site.xml`

Configure MapReduce to use YARN as the execution framework.

```xml
<configuration>

  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>

</configuration>
```

---

## `yarn-site.xml`

Configure the required NodeManager auxiliary service for MapReduce.

```xml
<configuration>

  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>

</configuration>
```

---

# 2. Environment Verification

Verify that the Hadoop and Hive environment variables are configured correctly.

```bash
echo $HADOOP_HOME
echo $HIVE_HOME
```

Expected output should point to the Hadoop and Hive installation directories.

---

# 3. PostgreSQL Hive Metastore

The Hive Metastore requires a PostgreSQL database to store Hive metadata.

## Create PostgreSQL User

Create a PostgreSQL user named `hive`.

```bash
sudo -u postgres createuser hive --password
```

Enter the desired password when prompted.

---

## Create Hive Metastore Database

Create the database and assign ownership to the `hive` user.

```bash
sudo -u postgres createdb hive_metastore -O hive
```

Connect to the database:

```bash
sudo -u postgres psql -d hive_metastore
```

---

## Configure Database Privileges

Execute the following SQL statements:

```sql
GRANT ALL PRIVILEGES ON DATABASE hive_metastore TO hive;

GRANT USAGE ON SCHEMA public TO hive;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO hive;

GRANT ALL
ON ALL SEQUENCES IN SCHEMA public
TO hive;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO hive;
```

---

## Initialize Hive Metastore Schema

Initialize the Hive schema for PostgreSQL.

```bash
sudo psql \
  -U postgres \
  -d hive_metastore \
  -a \
  -f /data/apache-archive/apache-hive-3.1.3-bin/scripts/metastore/upgrade/postgres/hive-schema-3.1.0.postgres.sql
```

> Ensure the database name matches the database configured in `hive-site.xml`.

---

# 4. PostgreSQL Authentication

Edit the PostgreSQL `pg_hba.conf` file.

Add the following configuration:

```conf
local   all             hive                                    md5
host    all             hive            127.0.0.1/32            md5
```

Reload or restart PostgreSQL after modifying the configuration.

```bash
sudo systemctl restart postgresql
```

---

# 5. Hive Configuration

## `hive-site.xml`

Configure Hive Metastore, HiveServer2, PostgreSQL, query optimization, and execution settings.

```xml
<configuration>

  <!-- PostgreSQL Metastore Connection -->

  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://10.10.10.105:5432/hive_metastore</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hive</value>
  </property>

  <property>
    <name>datanucleus.autoCreateSchema</name>
    <value>false</value>
  </property>


  <!-- Hive Metastore -->

  <property>
    <name>hive.metastore.thrift.bind.host</name>
    <value>10.10.10.105</value>
  </property>

  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://10.10.10.105:9083</value>
  </property>

  <property>
    <name>hive.metastore.client.socket.timeout</name>
    <value>300</value>
  </property>

  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
  </property>

  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>


  <!-- Query Optimization -->

  <property>
    <name>hive.auto.convert.join</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.auto.convert.join.noconditionaltask.size</name>
    <value>52428800</value>
  </property>

  <property>
    <name>hive.smbjoin.cache.rows</name>
    <value>10000</value>
  </property>

  <property>
    <name>mapred.reduce.tasks</name>
    <value>-1</value>
  </property>

  <property>
    <name>hive.exec.reducers.bytes.per.reducer</name>
    <value>67108864</value>
  </property>

  <property>
    <name>hive.exec.copyfile.maxsize</name>
    <value>33554432</value>
  </property>

  <property>
    <name>hive.exec.reducers.max</name>
    <value>1009</value>
  </property>


  <!-- Vectorized Execution -->

  <property>
    <name>hive.vectorized.groupby.checkinterval</name>
    <value>4096</value>
  </property>

  <property>
    <name>hive.vectorized.groupby.flush.percent</name>
    <value>0.1</value>
  </property>

  <property>
    <name>hive.compute.query.using.stats</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.vectorized.execution.enabled</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.vectorized.execution.reduce.enabled</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.vectorized.use.vectorized.input.format</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.vectorized.use.checked.expressions</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.vectorized.use.vector.serde.deserialize</name>
    <value>false</value>
  </property>

  <property>
    <name>hive.vectorized.adaptor.usage.mode</name>
    <value>chosen</value>
  </property>


  <!-- File Management -->

  <property>
    <name>hive.merge.mapfiles</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.merge.mapredfiles</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.merge.smallfiles.avgsize</name>
    <value>134217728</value>
  </property>

  <property>
    <name>hive.merge.size.per.task</name>
    <value>134217728</value>
  </property>


  <!-- Cost-Based Optimizer -->

  <property>
    <name>hive.cbo.enable</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.fetch.task.conversion</name>
    <value>more</value>
  </property>

  <property>
    <name>hive.fetch.task.conversion.threshold</name>
    <value>1073741824</value>
  </property>

  <property>
    <name>hive.limit.pushdown.memory.usage</name>
    <value>0.04</value>
  </property>


  <!-- Map-Side Aggregation -->

  <property>
    <name>hive.optimize.reducededuplication</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.optimize.reducededuplication.min.reducer</name>
    <value>4</value>
  </property>

  <property>
    <name>hive.map.aggr</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.map.aggr.hash.percentmemory</name>
    <value>0.5</value>
  </property>

  <property>
    <name>hive.optimize.sort.dynamic.partition</name>
    <value>false</value>
  </property>


  <!-- Hive Metastore Security and Events -->

  <property>
    <name>hive.metastore.execute.setugi</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.metastore.dml.events</name>
    <value>true</value>
  </property>

  <property>
    <name>hive.metastore.event.db.notification.api.auth</name>
    <value>false</value>
  </property>


  <!-- HiveServer2 -->

  <property>
    <name>hive.server2.authentication</name>
    <value>NONE</value>
  </property>

  <property>
    <name>hive.server2.thrift.bind.host</name>
    <value>0.0.0.0</value>
  </property>

  <property>
    <name>hive.server2.thrift.port</name>
    <value>9852</value>
  </property>

  <property>
    <name>hive.server2.transport.mode</name>
    <value>binary</value>
  </property>

  <property>
    <name>hive.server2.enable.doAs</name>
    <value>false</value>
  </property>


  <!-- Logging -->

  <property>
    <name>hive.root.logger</name>
    <value>WARN,DRFA</value>
  </property>

</configuration>
```

> **Security Note:** The password is stored in plain text in this example. For production environments, use a secure credential management mechanism.

---

# 6. Start Hive Services

## Start Hive Metastore

```bash
nohup /data/apache-archive/apache-hive-3.1.3-bin/bin/hive \
  --service metastore \
  > ~/metastore.log 2>&1 &
```

Verify the service:

```bash
ps -ef | grep HiveMetaStore
```

Check the log:

```bash
tail -f ~/metastore.log
```

---

## Start HiveServer2

```bash
nohup /data/apache-archive/apache-hive-3.1.3-bin/bin/hive \
  --service hiveserver2 \
  > ~/hiveserver2.log 2>&1 &
```

Verify the service:

```bash
ps -ef | grep HiveServer2
```

Check the log:

```bash
tail -f ~/hiveserver2.log
```

---

# 7. Connect Using Beeline

Connect to HiveServer2:

```bash
beeline -u jdbc:hive2://10.10.10.105:9852
```

If the connection is successful, Beeline should display the Hive prompt.

---

# 8. HDFS Data Lake Directory Structure

The following directory structure is used for the CDC Data Lake.

```text
/data-lake/
├── raw/
│   └── cdc/
├── staging/
│   └── cdc/
├── curated/
│   └── cdc/
└── logs/
    └── cdc/
```

Create the directories:

```bash
hdfs dfs -mkdir -p /data-lake/raw/cdc
hdfs dfs -mkdir -p /data-lake/staging/cdc
hdfs dfs -mkdir -p /data-lake/curated/cdc
hdfs dfs -mkdir -p /data-lake/logs/cdc
```

---

## Configure Ownership

```bash
hdfs dfs -chown -R hive:hive /data-lake/raw/cdc
hdfs dfs -chown -R hive:hive /data-lake/staging/cdc
hdfs dfs -chown -R hive:hive /data-lake/curated/cdc
hdfs dfs -chown -R hive:hive /data-lake/logs/cdc
```

---

## Configure Permissions

```bash
hdfs dfs -chmod -R 775 /data-lake/raw/cdc
hdfs dfs -chmod -R 775 /data-lake/staging/cdc
hdfs dfs -chmod -R 775 /data-lake/curated/cdc
hdfs dfs -chmod -R 775 /data-lake/logs/cdc
```

---

# 9. HDFS Write and Read Test

Write a simple test file:

```bash
echo "hello" | hdfs dfs -put - /data-lake/raw/cdc/test.txt
```

Verify the file:

```bash
hdfs dfs -cat /data-lake/raw/cdc/test.txt
```

Expected output:

```text
hello
```

---

# 10. Create CDC Database

Connect using Beeline and execute:

```sql
CREATE DATABASE IF NOT EXISTS cdc
LOCATION '/data-lake/raw/cdc';
```

Verify the database:

```sql
SHOW DATABASES;
```

Display database information:

```sql
DESCRIBE DATABASE cdc;
```

Use the database:

```sql
USE cdc;
```

---

# 11. Create External Table

## Raw Data Validation Table

Create a simple external table to verify that Hive can read data from HDFS.

```sql
CREATE EXTERNAL TABLE cdc.test_raw_check (
  dummy STRING
)
STORED AS TEXTFILE
LOCATION '/data-lake/raw/cdc';
```

Verify the table:

```sql
SHOW TABLES IN cdc;
```

---

## Customer CDC Table

Create an external Parquet table for CDC data.

```sql
USE cdc;

CREATE EXTERNAL TABLE customers (
  id STRING,
  name STRING
)
STORED AS PARQUET
LOCATION '/data-lake/raw/cdc/cdc.customers';
```

---

# 12. Verification Checklist

```text
[ ] HDFS is accessible.
[ ] YARN services are running.
[ ] Hadoop environment variables are configured.
[ ] PostgreSQL Hive Metastore database is accessible.
[ ] Hive Metastore is running on port 9083.
[ ] HiveServer2 is running on port 9852.
[ ] Beeline can connect successfully.
[ ] HDFS Data Lake directories are created.
[ ] HDFS permissions are configured correctly.
[ ] Hive CDC database is available.
[ ] Hive external tables can be created.
```

## Service Ports

| Service        | Port |
| -------------- | ---: |
| HDFS NameNode  | 8020 |
| Hive Metastore | 9083 |
| HiveServer2    | 9852 |
| PostgreSQL     | 5432 |


