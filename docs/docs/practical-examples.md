---
title: "Examples"
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance
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

# Examples

This guide provides practical guidance for common Iceberg Java API use cases and implementation patterns.

## Common Use Cases

### Table Operations

#### Creating Tables with Different Catalogs

**Hive Catalog**:
```java
import org.apache.iceberg.hive.HiveCatalog;
import org.apache.iceberg.catalog.Catalog;
import java.util.Map;
import java.util.HashMap;

HiveCatalog catalog = new HiveCatalog();
Map<String, String> properties = new HashMap<>();
properties.put("warehouse", "s3://your-bucket/warehouse");
properties.put("uri", "thrift://your-metastore:9083");

catalog.initialize("hive", properties);
```

**Hadoop Catalog**:
```java
import org.apache.iceberg.hadoop.HadoopCatalog;
import org.apache.hadoop.conf.Configuration;

Configuration conf = new Configuration();
String warehousePath = "hdfs://your-namenode:8020/warehouse";
HadoopCatalog catalog = new HadoopCatalog(conf, warehousePath);
```

#### Reading and Writing Data

**Append Data**:
```java
import org.apache.iceberg.data.GenericRecord;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.io.DataFile;
import org.apache.iceberg.catalog.TableIdentifier;

// Create data records
List<Record> records = new ArrayList<>();
// ... populate records with your data

// Append to table
Table table = catalog.loadTable(TableIdentifier.of("database", "table"));
DataFile dataFile = table.newAppend()
    .appendFile(yourDataFile)
    .commit();
```

**Overwrite Data**:
```java
Table table = catalog.loadTable(TableIdentifier.of("database", "table"));
table.newOverwrite()
    .addFile(newDataFile)
    .deleteFile(oldDataFile)
    .commit();
```

### Schema Evolution

#### Adding Columns
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.types.Types;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));

table.updateSchema()
    .addColumn("new_column", Types.StringType.get())
    .commit();
```

#### Renaming Columns
```java
table.updateSchema()
    .renameColumn("old_name", "new_name")
    .commit();
```

#### Changing Column Types
```java
table.updateSchema()
    .modifyColumn("column_name", Types.IntegerType.get())
    .commit();
```

### Partition Evolution

#### Adding New Partition Field
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));

table.updateSpec()
    .addField("new_partition_column")
    .commit();
```

#### Removing Partition Field
```java
table.updateSpec()
    .removeField("partition_column")
    .commit();
```

### Time Travel Operations

#### Query by Snapshot ID
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.io.CloseableIterable;
import org.apache.iceberg.data.Record;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));
long snapshotId = table.history().get(0).snapshotId(); // Get from table history

TableScan scan = table.newScan()
    .useSnapshotId(snapshotId);

try (CloseableIterable<Record> results = scan.open()) {
    for (Record record : results) {
        // Process record
    }
}
```

#### Query by Timestamp
```java
import java.util.concurrent.TimeUnit;

// Query data as of 24 hours ago
long timestampMillis = System.currentTimeMillis() - TimeUnit.HOURS.toMillis(24);

TableScan scan = table.newScan()
    .asOfTime(timestampMillis);
```

#### Rollback to Previous Snapshot
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));
long snapshotId = table.history().get(0).snapshotId();

table.manageSnapshots()
    .rollbackTo(snapshotId)
    .commit();
```

### Branching and Tagging

#### Create Branch
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;
import java.util.concurrent.TimeUnit;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));

table.manageSnapshots()
    .createBranch("test-branch", table.currentSnapshot().snapshotId())
    .setMinSnapshotsToKeep("test-branch", 2)
    .setMaxSnapshotAgeMs("test-branch", TimeUnit.HOURS.toMillis(1)) // 1 hour
    .setMaxRefAgeMs("test-branch", TimeUnit.DAYS.toMillis(7)) // 1 week
    .commit();
```

#### Create Tag
```java
table.manageSnapshots()
    .createTag("v1.0.0", table.currentSnapshot().snapshotId())
    .setMaxRefAgeMs("v1.0.0", TimeUnit.DAYS.toMillis(365)) // 1 year
    .commit();
```

#### Write to Branch
```java
table.newAppend()
    .appendFile(yourDataFile)
    .toBranch("test-branch")
    .commit();
```

### Data Management

#### Expire Snapshots
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;
import java.util.concurrent.TimeUnit;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));

table.expireSnapshots()
    .expireOlderThan(System.currentTimeMillis() - TimeUnit.DAYS.toMillis(1)) // 24 hours
    .retainLast(5)
    .commit();
```

#### Rewrite Files
```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;
import com.google.common.collect.ImmutableSet;
import org.apache.iceberg.io.DataFile;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));

table.newRewrite()
    .rewriteFiles(ImmutableSet.of(smallFile1, smallFile2), 
                 ImmutableSet.of(compactedFile))
    .commit();
```

## Implementation Patterns

### Batch Processing Pattern

This pattern demonstrates how to efficiently write data in batches to optimize performance and reduce commit overhead.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.io.DataFile;
import java.util.List;
import java.util.ArrayList;

public class IcebergBatchWriter {
    private final Table table;
    private final int batchSize;
    private final List<Record> batch;
    
    public IcebergBatchWriter(Table table, int batchSize) {
        this.table = table;
        this.batchSize = batchSize;
        this.batch = new ArrayList<>(batchSize);
    }
    
    public void write(Record record) {
        batch.add(record);
        if (batch.size() >= batchSize) {
            flush();
        }
    }
    
    public void flush() {
        if (!batch.isEmpty()) {
            // Write batch to Iceberg
            DataFile dataFile = writeBatch(batch);
            table.newAppend().appendFile(dataFile).commit();
            batch.clear();
        }
    }
    
    public void close() {
        flush();
    }
    
    private DataFile writeBatch(List<Record> batch) {
        // Implement your batch writing logic here
        // This would typically convert records to a DataFile
        return null;
    }
}
```

### Stream Processing Pattern

This pattern demonstrates how to handle streaming data by managing file sizes and creating new files as needed.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.io.DataFile;

public class IcebergStreamWriter {
    private final Table table;
    private final long maxFileSize;
    private long currentFileSize = 0;
    private DataFile currentFile;
    
    public IcebergStreamWriter(Table table, long maxFileSize) {
        this.table = table;
        this.maxFileSize = maxFileSize;
    }
    
    public void write(Record record) {
        if (currentFile == null || currentFileSize > maxFileSize) {
            if (currentFile != null) {
                table.newAppend().appendFile(currentFile).commit();
            }
            currentFile = createNewFile();
            currentFileSize = 0;
        }
        
        // Write record to current file
        writeToFile(currentFile, record);
        currentFileSize += recordSize(record);
    }
    
    private DataFile createNewFile() {
        // Implement file creation logic
        return null;
    }
    
    private void writeToFile(DataFile file, Record record) {
        // Implement record writing logic
    }
    
    private long recordSize(Record record) {
        // Calculate record size
        return 0;
    }
}
```

### Schema Migration Pattern

This pattern provides a safe way to migrate schemas with validation before applying changes.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.Schema;

public class SchemaMigrator {
    public static void migrateSchema(Table table, Schema newSchema) {
        Schema currentSchema = table.schema();
        
        // Validate schema evolution
        if (!isValidEvolution(currentSchema, newSchema)) {
            throw new IllegalArgumentException("Invalid schema evolution");
        }
        
        // Apply schema changes
        table.updateSchema()
            .unionSchemaWith(newSchema)
            .commit();
    }
    
    private static boolean isValidEvolution(Schema oldSchema, Schema newSchema) {
        // Implement validation logic
        // Check for compatible type changes, required field additions, etc.
        return true;
    }
}
```

### Partition Strategy Pattern

This pattern helps optimize partition specs based on query patterns and data characteristics.

```java
import org.apache.iceberg.PartitionSpec;
import org.apache.iceberg.Schema;
import org.apache.iceberg.types.Types;
import java.util.List;

public class PartitionOptimizer {
    public static PartitionSpec optimizePartitionSpec(Schema schema, 
                                                      List<String> frequentColumns) {
        PartitionSpec.Builder builder = PartitionSpec.builderFor(schema);
        
        for (String column : frequentColumns) {
            Types.NestedField field = schema.findField(column);
            if (field != null) {
                if (field.type() == Types.TimestampType.withZone()) {
                    builder.day(column); // Partition by day for timestamps
                } else if (field.type().isPrimitiveType()) {
                    builder.identity(column); // Identity transform for other primitives
                }
            }
        }
        
        return builder.build();
    }
}
```

## Performance Optimization

### Scan Planning

Use row filtering and projection pushdown to reduce the amount of data scanned and improve query performance.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.TableScan;
import org.apache.iceberg.expressions.Expressions;

// Use row filtering to reduce data scanned
TableScan scan = table.newScan()
    .filter(Expressions.equal("status", "active"))
    .select("id", "name", "status");
```

### File Size Optimization

Configure target file size to optimize for your workload. Larger files reduce metadata overhead but may impact parallelism.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.catalog.TableIdentifier;

Table table = catalog.loadTable(TableIdentifier.of("database", "table"));
table.updateProperties()
    .set("write.target-file-size-bytes", "536870912") // 512 MB in bytes
    .commit();
```

### Compression Configuration

Configure Parquet compression to reduce storage costs while balancing with compression speed.

```java
table.updateProperties()
    .set("write.parquet.compression-codec", "zstd")
    .set("write.parquet.compression-level", "3") // 1-9, higher = better compression but slower
    .commit();
```

## Error Handling

### Concurrent Write Handling

Implement retry logic with exponential backoff to handle concurrent modification failures gracefully.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.io.DataFile;
import org.apache.iceberg.exceptions.CommitFailedException;

int maxRetries = 3;
for (int i = 0; i < maxRetries; i++) {
    try {
        table.newAppend()
            .appendFile(yourDataFile)
            .commit();
        break; // Success, exit retry loop
    } catch (CommitFailedException e) {
        if (i == maxRetries - 1) {
            throw e; // Final retry failed, propagate exception
        }
        // Implement exponential backoff
        long waitTime = (long) Math.pow(2, i) * 1000; // 1s, 2s, 4s
        Thread.sleep(waitTime);
    }
}
```

### Schema Validation

Validate records against the table schema before writing to ensure data quality and prevent write failures.

```java
import org.apache.iceberg.Schema;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.types.Types;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public void validateRecordAgainstSchema(Record record, Schema schema) {
    Logger LOG = LoggerFactory.getLogger(getClass());
    
    try {
        // Validate record structure
        for (Types.NestedField field : schema.columns()) {
            Object value = record.get(field.name());
            if (value != null && !isValidType(value, field.type())) {
                throw new IllegalArgumentException(
                    "Invalid type for field: " + field.name());
            }
        }
    } catch (IllegalArgumentException e) {
        LOG.error("Schema validation failed", e);
        throw e;
    }
}

private boolean isValidType(Object value, org.apache.iceberg.types.Type type) {
    // Implement type validation logic
    return true;
}
```

## Best Practices

### Resource Management

Always use try-with-resources for Iceberg operations to ensure proper resource cleanup and prevent resource leaks.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.TableScan;
import org.apache.iceberg.io.CloseableIterable;
import org.apache.iceberg.data.Record;

try (TableScan scan = table.newScan()) {
    try (CloseableIterable<Record> results = scan.open()) {
        for (Record record : results) {
            // Process records
        }
    }
}
```

### Catalog Configuration

Use appropriate catalog for your use case:
- **Hive Catalog**: For production with Hive metastore
- **Hadoop Catalog**: For HDFS-based deployments without metastore
- **Custom Catalog**: For cloud-specific implementations

### Partition Design

Choose partition columns carefully:
- Use high-cardinality columns for identity partitioning
- Use time-based partitioning for timestamp columns
- Avoid over-partitioning (too many small partitions)

### Schema Evolution

Follow schema evolution best practices:
- Always add columns as optional first
- Use column IDs for compatibility
- Test schema changes in development first

## Testing Patterns

### Table Test Utilities

Utility methods to simplify testing with Iceberg tables.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.Schema;
import org.apache.iceberg.catalog.Catalog;
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.data.Record;
import org.apache.iceberg.io.CloseableIterable;
import java.util.List;
import java.util.ArrayList;

public class IcebergTestUtils {
    public static Table createTestTable(Catalog catalog, String name, Schema schema) {
        TableIdentifier identifier = TableIdentifier.of("test", name);
        return catalog.createTable(identifier, schema);
    }
    
    public static void dropTestTable(Catalog catalog, String name) {
        TableIdentifier identifier = TableIdentifier.of("test", name);
        catalog.dropTable(identifier);
    }
    
    public static List<Record> readTable(Table table) {
        List<Record> records = new ArrayList<>();
        try (TableScan scan = table.newScan();
             CloseableIterable<Record> results = scan.open()) {
            for (Record record : results) {
                records.add(record);
            }
        }
        return records;
    }
}
```

## Integration Examples

### Apache Spark Integration

Configure Spark to use Iceberg as the default catalog for optimal integration.

```java
import org.apache.spark.sql.SparkSession;

SparkSession spark = SparkSession.builder()
    .config("spark.sql.catalog.my_catalog", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.my_catalog.type", "hadoop")
    .config("spark.sql.catalog.my_catalog.warehouse", "s3://your-bucket/warehouse")
    .getOrCreate();

// Use Iceberg table in Spark
spark.table("my_catalog.database.table").show();
```

### Apache Flink Integration

Use Flink's Iceberg sink for streaming data processing with exactly-once semantics.

```java
import org.apache.iceberg.flink.TableLoader;
import org.apache.iceberg.flink.sink.FlinkSink;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.types.Row;

TableLoader tableLoader = TableLoader.fromHadoopTable("hdfs://your-namenode:8020/warehouse/database/table");

DataStream<Row> stream = ... // Your data stream

FlinkSink.forRow(stream)
    .tableLoader(tableLoader)
    .append();
```

## Getting Help

- **Documentation**: Check the [Java API documentation](java-api-quickstart.md)
- **Configuration**: Review [configuration options](configuration.md)
- **Community**: Join the [Apache Iceberg community](https://iceberg.apache.org/community/)
- **Issues**: Report bugs on [GitHub Issues](https://github.com/apache/iceberg/issues)
