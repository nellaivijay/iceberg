---
title: "Troubleshooting"
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may use this file except in compliance
 - with the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
-->

# Troubleshooting

This guide covers common issues and solutions when working with Apache Iceberg using the Java API.

## Catalog Issues

### Hive Catalog Connection Failures

**Problem**: Unable to connect to Hive Metastore

**Symptoms**:
- `MetaException: Unable to connect to metastore`
- Connection timeout errors

**Solutions**:

1. **Check Metastore Configuration**:
```java
Map<String, String> properties = new HashMap<>();
properties.put("uri", "thrift://your-metastore:9083");
properties.put("warehouse", "s3://your-bucket/warehouse");

catalog.initialize("hive", properties);
```

2. **Verify Network Connectivity**:
```bash
telnet your-metastore 9083
```

3. **Check Metastore Logs**:
```bash
tail -f /var/log/hive/metastore.log
```

4. **Validate Kerberos Configuration** (if using Kerberos):
```java
System.setProperty("java.security.krb5.conf", "/path/to/krb5.conf");
UserGroupInformation.setConfiguration(conf);
```

### Hadoop Catalog Permission Issues

**Problem**: Permission denied when accessing HDFS

**Symptoms**:
- `Permission denied` errors
- Unable to create tables in warehouse

**Solutions**:

1. **Check HDFS Permissions**:
```bash
hdfs dfs -ls /warehouse
hdfs dfs -chmod 755 /warehouse
```

2. **Verify User Impersonation**:
```java
Configuration conf = new Configuration();
conf.set("hadoop.proxy.user.yarn.hosts", "*");
conf.set("hadoop.proxy.user.yarn.groups", "*");
```

3. **Check Hadoop Configuration**:
```java
Configuration conf = new Configuration();
conf.addResource(new Path("/etc/hadoop/core-site.xml"));
conf.addResource(new Path("/etc/hadoop/hdfs-site.xml"));
```

## Table Operations

### Concurrent Write Failures

**Problem**: `CommitFailedException` when multiple writers attempt to modify the same table

**Symptoms**:
- `CommitFailedException: Commit failed`
- Concurrent modification detected

**Solutions**:

1. **Implement Retry Logic**:
```java
int maxRetries = 3;
for (int i = 0; i < maxRetries; i++) {
    try {
        table.newAppend()
            .appendFile(FILE_A)
            .commit();
        break;
    } catch (CommitFailedException e) {
        if (i == maxRetries - 1) {
            throw e;
        }
        Thread.sleep(1000 * (i + 1)); // Exponential backoff
    }
}
```

2. **Use Write-Audit-Publish (WAP)**:
```java
table.updateProperties()
    .set("write.wap.enabled", "true")
    .commit();

table.newAppend()
    .appendFile(FILE_A)
    .toBranch("wap-branch")
    .commit();
```

3. **Optimize Write Distribution**:
```java
table.updateProperties()
    .set("write.distribution-mode", "hash")
    .commit();
```

### Schema Evolution Conflicts

**Problem**: Schema evolution operations fail due to incompatible changes

**Symptoms**:
- `IllegalArgumentException: Cannot add required column`
- Schema validation errors

**Solutions**:

1. **Add Columns as Optional**:
```java
// Instead of this (will fail):
table.updateSchema()
    .addColumn("new_column", Types.StringType.get(), "description") // Required
    .commit();

// Use this:
table.updateSchema()
    .addColumn("new_column", Types.StringType.get(), "description", "default_value") // Optional
    .commit();
```

2. **Validate Schema Changes**:
```java
Schema newSchema = table.schema()
    .union(Schema.builder()
        .addColumn("new_column", Types.StringType.get())
        .build());

if (table.schema().sameSchema(newSchema)) {
    LOG.info("Schema unchanged, skipping update");
} else {
    table.updateSchema()
        .unionSchemaWith(newSchema)
        .commit();
}
```

3. **Use Column IDs for Compatibility**:
```java
table.updateSchema()
    .addColumn("new_column", Types.StringType.get())
    .setIdentifierFields(Arrays.asList("id")) // Set identifier fields
    .commit();
```

## Performance Issues

### Slow Query Performance

**Problem**: Queries take longer than expected

**Symptoms**:
- High query latency
- Excessive file scanning

**Solutions**:

1. **Optimize Partitioning**:
```java
// Analyze query patterns and add appropriate partitions
table.updateSpec()
    .addField("date_column") // Partition by date
    .addField("region_column") // Partition by region
    .commit();
```

2. **Configure File Size**:
```java
table.updateProperties()
    .set("write.target-file-size-bytes", "536870912") // 512 MB
    .commit();
```

3. **Enable Statistics Collection**:
```java
table.updateProperties()
    .set("write.metadata.metrics.default", "truncate(16)")
    .set("write.metadata.metrics.column.my_column", "full")
    .commit();
```

4. **Use Projection Pushdown**:
```java
TableScan scan = table.newScan()
    .select("id", "name", "date") // Only select needed columns
    .filter(Expressions.equal("status", "active"));
```

### Memory Issues

**Problem**: OutOfMemoryError during operations

**Symptoms**:
- `OutOfMemoryError: Java heap space`
- Process crashes during large operations

**Solutions**:

1. **Increase JVM Heap Size**:
```bash
java -Xmx8g -Xms8g YourApplication
```

2. **Process Data in Batches**:
```java
int batchSize = 10000;
List<Record> batch = new ArrayList<>(batchSize);

for (Record record : records) {
    batch.add(record);
    if (batch.size() >= batchSize) {
        writeBatch(batch);
        batch.clear();
    }
}
```

3. **Configure Split Size**:
```java
table.updateProperties()
    .set("read.split.target-size", "134217728") // 128 MB
    .commit();
```

4. **Use Streaming Reads**:
```java
try (TableScan scan = table.newScan();
     CloseableIterable<Record> results = scan.open()) {
    for (Record record : results) {
        // Process one record at a time
    }
}
```

## Storage Issues

### S3 Connection Problems

**Problem**: Unable to connect to S3 or other cloud storage

**Symptoms**:
- `AmazonClientException: Unable to execute HTTP request`
- Connection timeout errors

**Solutions**:

1. **Configure S3 Credentials**:
```java
Configuration conf = new Configuration();
conf.set("fs.s3a.access.key", "your-access-key");
conf.set("fs.s3a.secret.key", "your-secret-key");
conf.set("fs.s3a.endpoint", "s3.amazonaws.com");
```

2. **Use IAM Role**:
```java
conf.set("fs.s3a.aws.credentials.provider", 
         "com.amazonaws.auth.InstanceProfileCredentialsProvider");
```

3. **Configure Endpoint for Non-AWS S3**:
```java
conf.set("fs.s3a.endpoint", "https://s3.example.com");
conf.set("fs.s3a.path.style.access", "true");
```

4. **Enable SSL**:
```java
conf.set("fs.s3a.connection.ssl.enabled", "true");
```

### Metadata File Bloat

**Problem**: Too many metadata files causing performance degradation

**Symptoms**:
- Slow metadata operations
- Excessive metadata file count

**Solutions**:

1. **Expire Old Snapshots**:
```java
import java.util.concurrent.TimeUnit;

table.expireSnapshots()
    .expireOlderThan(System.currentTimeMillis() - TimeUnit.DAYS.toMillis(1)) // 24 hours
    .retainLast(5)
    .commit();
```

2. **Enable Metadata Compression**:
```java
table.updateProperties()
    .set("write.metadata.compression-codec", "gzip")
    .commit();
```

3. **Configure Metadata Cleanup**:
```java
table.updateProperties()
    .set("write.metadata.delete-after-commit.enabled", "true")
    .set("write.metadata.previous-versions-max", "100")
    .commit();
```

4. **Run Rewrite Manifests**:
```java
table.rewriteManifests()
    .rewriteIf(file -> file.length() < 1024 * 1024) // Rewrite small manifests
    .commit();
```

## Time Travel Issues

### Invalid Snapshot ID

**Problem**: Attempting to query a non-existent snapshot

**Symptoms**:
- `IllegalArgumentException: Snapshot not found`
- Time travel query fails

**Solutions**:

1. **Validate Snapshot ID**:
```java
long snapshotId = table.currentSnapshot().snapshotId(); // Get current snapshot
if (table.snapshot(snapshotId) == null) {
    throw new IllegalArgumentException("Snapshot not found: " + snapshotId);
}

TableScan scan = table.newScan()
    .useSnapshotId(snapshotId);
```

2. **Use Available Snapshot**:
```java
long snapshotId = table.currentSnapshot().snapshotId();
if (snapshotId == -1) {
    throw new IllegalStateException("No snapshots available");
}
```

3. **List Available Snapshots**:
```java
for (Snapshot snapshot : table.snapshots()) {
    System.out.println("Snapshot ID: " + snapshot.snapshotId());
    System.out.println("Timestamp: " + snapshot.timestampMs());
}
```

### Branch/Tag Not Found

**Problem**: Attempting to use a non-existent branch or tag

**Symptoms**:
- `IllegalArgumentException: Ref not found`
- Branch/tag reference fails

**Solutions**:

1. **Check if Branch Exists**:
```java
SnapshotRef ref = table.ref("my-branch");
if (ref == null) {
    throw new IllegalArgumentException("Branch not found: my-branch");
}
```

2. **Create Branch if Missing**:
```java
if (table.ref("my-branch") == null) {
    table.manageSnapshots()
        .createBranch("my-branch", table.currentSnapshot().snapshotId())
        .commit();
}
```

3. **List Available References**:
```java
Map<String, SnapshotRef> refs = table.refs();
refs.forEach((name, ref) -> {
    System.out.println("Ref: " + name + ", Type: " + ref.type());
});
```

## Data Quality Issues

### Schema Mismatch

**Problem**: Data files don't match table schema

**Symptoms**:
- `Schema mismatch` errors during reads
- Invalid data type errors

**Solutions**:

1. **Validate Data Before Writing**:
```java
public void validateRecord(Record record, Schema schema) {
    for (Types.NestedField field : schema.columns()) {
        Object value = record.get(field.name());
        if (value != null && !isValidType(value, field.type())) {
            throw new IllegalArgumentException(
                "Invalid type for field: " + field.name());
        }
    }
}
```

2. **Use Schema Evolution**:
```java
table.updateSchema()
    .addColumn("new_column", Types.StringType.get())
    .commit();
```

3. **Handle Null Values**:
```java
// Make column optional if null values are expected
table.updateSchema()
    .makeColumnOptional("column_name")
    .commit();
```

### Duplicate Records

**Problem**: Duplicate records in table

**Symptoms**:
- Incorrect query results
- Data integrity issues

**Solutions**:

1. **Use Deduplication**:
```java
// In Spark
spark.table("my_catalog.db.table")
    .dropDuplicates("id")
    .writeTo("my_catalog.db.table")
    .append();
```

2. **Implement Idempotent Writes**:
```java
// Check if record exists before writing
if (!recordExists(record.getId())) {
    table.newAppend()
        .appendFile(createDataFile(record))
        .commit();
}
```

3. **Use Primary Key Constraints** (if supported):
```java
// Configure table with unique constraints
table.updateProperties()
    .set("write.upsert.enabled", "true")
    .commit();
```

## Configuration Issues

### Property Not Recognized

**Problem**: Table property not recognized or not taking effect

**Symptoms**:
- Property ignored
- No effect on behavior

**Solutions**:

1. **Check Property Spelling**:
```java
// Correct
table.updateProperties()
    .set("write.target-file-size-bytes", "536870912")
    .commit();

// Incorrect (missing hyphens)
table.updateProperties()
    .set("write.targetfilesizebytes", "536870912")
    .commit();
```

2. **Validate Property Value**:
```java
String property = "write.target-file-size-bytes";
String value = "invalid";

try {
    Long.parseLong(value);
} catch (NumberFormatException e) {
    throw new IllegalArgumentException("Invalid value for property: " + property);
}
```

3. **Check Catalog-Level Properties**:
```java
// Some properties must be set at catalog level
catalog.updateProperties("db", "table")
    .set("catalog-level-property", "value")
    .commit();
```

### Version Compatibility

**Problem**: Incompatible Iceberg versions between components

**Symptoms**:
- `UnsupportedOperationException`
- Version mismatch errors

**Solutions**:

1. **Check Iceberg Version**:
```java
import org.apache.iceberg.IcebergBuild;

System.out.println("Iceberg version: " + IcebergBuild.version());
```

2. **Ensure Compatible Versions**:
```xml
<!-- In pom.xml -->
<dependency>
    <groupId>org.apache.iceberg</groupId>
    <artifactId>iceberg-core</artifactId>
    <version>1.5.0</version>
</dependency>
```

3. **Upgrade Gradually**:
- Upgrade catalog first
- Then upgrade tables
- Finally upgrade clients

## Debugging Tips

### Enable Debug Logging

```java
// Enable Iceberg debug logging
System.setProperty("org.apache.iceberg", "DEBUG");

// Or use Log4j configuration
<logger name="org.apache.iceberg" level="DEBUG"/>
```

### Inspect Table Metadata

```java
Table table = catalog.loadTable(TableIdentifier.of("db", "table"));

// Print table schema
System.out.println("Schema: " + table.schema());

// Print partition spec
System.out.println("Partition spec: " + table.spec());

// Print table properties
System.out.println("Properties: " + table.properties());

// Print snapshot history
for (Snapshot snapshot : table.history()) {
    System.out.println("Snapshot: " + snapshot.snapshotId() + 
                      ", Time: " + snapshot.timestampMs());
}
```

### Profile Query Performance

```java
long startTime = System.currentTimeMillis();

try (TableScan scan = table.newScan();
     CloseableIterable<Record> results = scan.open()) {
    for (Record record : results) {
        // Process records
    }
}

long endTime = System.currentTimeMillis();
System.out.println("Query time: " + (endTime - startTime) + "ms");
```

## Getting Help

If you encounter issues not covered here:

1. **Check Documentation**: Review [API documentation](java-api-quickstart.md) and [configuration](configuration.md)
2. **Search Issues**: Look for similar issues on [GitHub Issues](https://github.com/apache/iceberg/issues)
3. **Community Support**: Join the [Apache Iceberg community](https://iceberg.apache.org/community/)
4. **Create Minimal Reproducible Example**: When reporting issues, include:
   - Iceberg version
   - Catalog type and configuration
   - Table schema and properties
   - Minimal code example
   - Error messages and stack traces
5. **Enable Debug Logging**: Provide debug logs when reporting issues
