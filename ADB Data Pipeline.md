# ADB Data Pipelines

- Created: `=dateformat(this.file.ctime, "DDDD, HH:mm")`
- Last Modified: `=dateformat(this.file.mtime, "DDDD, HH:mm")`
- Tags: #IT #Oracle #AutonomousDB
---
## Concepts

[Resource Principal](Resource%20Principal.md)

### 1.1 Automated Data Transfer between Object Store and Autonomous Database
- Normally we use `DBMS_CLOUD.COPY_DATA` for loading and `DBMS_CLOUD.EXPORT_DATA` for exporting.
- Data Pipeline allows us to automate these manual processes through `DBMS_CLOUD_PIPELINE` interface.
    - Automated load: newly arrived object files
        - Usage
            - Automatic data loading
            - Incremental data migration
    - Automated export: newly inserted data
        - Usage
            - Automatic sharing of new database data
            - Automatic sharing of some logs; these are built-in (preconfigured) and called *Oracle-maintained pipelines*
                - DB audit log (`ORA$AUDIT_EXPORT`)
                - APEX workspace activity log (`ORA$APEX_ACTIVITY_LOG`)
- However the automation is done in an asynchronous way, because it's based on DB scheduler job.

<br/>

### 1.2. Lifecycle of Data Pipelines

- ① Create -> ② Configure -> ③ Test (and reset) -> ④ Start / Stop / Drop
    - Each step matches a certain procedure of `DBMS_CLOUD_PIPELINE`
    - Besides, various dictionary views are provided for monitoring/troubleshooting purpose
        - Pipeline itself: `XXX_CLOUD_PIPELINES`
            - `STATUS_TABLE` column enables additional monitoring for load pipelines
                - `STATUS_TABLE.OPERATION_ID` -> `XXX_LOAD_OPERATIONS.ID`
        - Pipeline Attributes: `XXX_CLOUD_PIPELINE_ATTRIBUTES`
        - Pipeline status: `XXX_CLOUD_PIPELINE_HISTORY`
    - Oracle-maintained pipelines are initially in `STOPPED` status. To use them, (modify the configuration if needed and) start them as `ADMIN` user.

<br/>

### 1.3. Additional Characteristics/Features

- Atomicity: "exactly once semantic"
    - Strictly file name based: nothing happens when content changes after loading or files are deleted.
- Recoverbility
    - Automatic retry on next run schedule.
    - Failure on some files doesn't stop the pipeline.
- Has the same support scope as `DBMS_CLOUD`.
    - Various object stores (credential & URI format)
        - OCI, Amazon S3, Google Cloud Storage, Azure Blob Storage, Wasabi
    - Various formats
        - CSV, JSON, XML, Parquet, ORC, AVRO

<br/>

## 2. Demo

### 2.1. Prerequisites

#### 2.1.1. Basic Environments

- Prepare both Oracle client and OCI-CLI
- Provision an ADW instance (`MyADW`)
- Create an object bucket: `pipeline_bucket`
    - Create folders: `demo_load`, `demo_export`
        - One pipeline is needed for each table. So using a single bucket alone is not so good idea.

<br/>

#### 2.1.2. Resource Principal for the ADW Instance

> Using resource principal is a bit more elegant way of creating credentials in ADB than "borrowing" an OCI user principal

- Create a dynamic Group: `sylee_dgroup`
    - Matching Rule: `resource.id = '<OCID of MyADW>'`
- Create a policy: `sylee_dgroup_policy`
    - `allow dynamic-group sylee_dgroup to manage object-family in compartment sangyun.lee`
- Enable resource principal for `ADMIN` user
    - `execute dbms_cloud_admin.enable_resource_principal()`
- As a result, credential `OCI$RESOURCE_PRINCIPAL` gets created (can be seen in `DBA_CREDENTIALS`)

<br/>

#### 2.1.3. Table and Data

- Create the demo table
    - This will be used for both loading and exporting demo.
    - For exporting, the `ts` column will be the `key_column`, which is used to identify new data in the table.

```sql
create table demo (
    name varchar2(10),
    val  number,
    ts   timestamp default systimestamp
);
```

- Insert some data into the demo table
    - Save this as `insert.sql` so that we can also use it in the export demo.

```sql
insert into demo (name, val)
    select 'AAAAAAAAAA', 10000
      from dual
    connect by level <= 10;
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
ddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
commit;
```

- Unload the table data as a csv file.
    - This should be executed at SQL*Plus
    - Use "YYYY-MM-DD HH24:MI:SS.FF6" as timestamp format (`NLS_TIMESTAMP_FORMAT`)

```sql
set echo off
set heading off feedback off timing off
set linesize 10000 pagesize 0
set trims on trimspool on
set serveroutput off termout off
set markup csv on delimiter | quote off
spool demo.csv

select * from demo;

spool off
```

> Or 10-row file can be just fabricated from scratch.

<br/>

### 2.2. Data Pipeline for Loading

#### 2.2.1. Create the Pipeline

```sql
begin
    dbms_cloud_pipeline.create_pipeline(
        pipeline_name => 'PIPELINE_LOAD_DEMO',
        pipeline_type => 'LOAD'
    );
end;
/
```

<br/>

#### 2.2.2. Configure the Pipeline

- Now set the attributes of the pipeline
    - By the way, we can configure the pipeline while creating it.

```sql
begin
    dbms_cloud_pipeline.set_attribute(
        pipeline_name => 'PIPELINE_LOAD_DEMO',
        attributes    => json_object(
            'credential_name' value 'OCI$RESOURCE_PRINCIPAL',
            'location'        value 'https://objectstorage.ap-seoul-1.oraclecloud.com/n/apackrsct01/b/pipeline_bucket/o/demo_load/',
            'table_name'      value 'DEMO',
            'format'          value '{"type": "csv", "delimiter": "|", "timestampformat": "YYYY-MM-DD HH24:MI:SS.FF6"}',
            'priority'        value 'LOW',
            'interval'        value '1')
    );
end;
/
```

- The pipeline will load data from `demo_load` folder (`location`)
- The timestamp format must match the one used when unloading. (`format`)
- For small demo like this, low service (`priority`) and 1 minute cycle (`interval`) are enough.

<br/>

#### 2.2.3. Test

- First reset the demo table

```sql
truncate table demo;
```

- Upload the (unloaded) demo.csv file to Object Store

```zsh
oci os object put --file demo.csv --name demo_load/demo1.csv

# check the result
oci os object ls
```

> A little simplified commands thanks to `oci_cli_rc` setting

- Now test-run
    - 10 rows will be loaded immediately. Check the result by querying demo table.

```sql
begin
    dbms_cloud_pipeline.run_pipeline_once(
        pipeline_name => 'PIPELINE_LOAD_DEMO'
    );
end;
/
```

- Reset the Pipeline
    - After test run, the pipeline must be reset.
    - At the same time we can delete the table data (`purge_data => true`).

```sql
begin
    dbms_cloud_pipeline.reset_pipeline(
        pipeline_name => 'PIPELINE_LOAD_DEMO',
        purge_data => true
    );
end;
/
```

<br/>

#### 2.2.4. Start the Pipeline

```sql
begin
    dbms_cloud_pipeline.start_pipeline(
        pipeline_name => 'PIPELINE_LOAD_DEMO'
    );
end;
/
```

- This starts the work as a scheduled job
- Again, 10 rows are loaded from the already uploaded object `demo1.csv`

<br/>

#### 2.2.5. Final Test: Arrival of New File(s) at Object Store

- First, upload the demo.csv file as demo1.csv object again
    - No more load will be performed due to "atomicity", remember?

```zsh
oci os object put --file demo.csv --name demo_load/demo1.csv --force
```

- Now try a different name

```zsh
oci os object put --file demo.csv --name demo_load/demo2.csv
```

- After a while (in less than 1 minute at top), we can see 10 more rows were loaded. The data pipeline works!

<br/>

#### 2.2.6. Clean-up: Stop and Drop the Pipeline

```sql
begin
    dbms_cloud_pipeline.stop_pipeline(pipeline_name => 'PIPELINE_LOAD_DEMO');
    dbms_cloud_pipeline.drop_pipeline(pipeline_name => 'PIPELINE_LOAD_DEMO');
end;
/
```

<br/>

### 2.3. Data Pipeline for Exporting

#### 2.3.1. Create the Pipeline

```sql
begin
    dbms_cloud_pipeline.create_pipeline(
        pipeline_name => 'PIPELINE_EXPORT_DEMO',
        pipeline_type => 'EXPORT'
    );
end;
/
```

<br/>

#### 2.3.2. Configure the Pipeline

```sql
begin
    dbms_cloud_pipeline.set_attribute(
        pipeline_name => 'PIPELINE_EXPORT_DEMO',
        attributes    => json_object(
            'credential_name' value 'OCI$RESOURCE_PRINCIPAL',
            'location'        value 'https://objectstorage.ap-seoul-1.oraclecloud.com/n/apackrsct01/b/pipeline_bucket/o/demo_export/',
            'table_name'      value 'DEMO',
            'key_column'      value 'TS',
            'format'          value '{"type": "csv", "delimiter": "|"}',
            'priority'        value 'LOW',
            'interval'        value '1')
    );
end;
/
```

- The exported file will be created in `demo_export` folder (`location`)
- Set the table name. We can also use `query` parameter instead of `table_name`
- The "key column" is `TS`.
    - We can also choose not to specify the key column. In this case, the incremental export should be manually implemented using the `where` clause of the `query` parameter. (But what's the point of doing it?)
- Timestamp format is not specified. In fact, "YYYY-MM-DD HH24:MI:SS.FF6" is the default format.

<br/>

#### 2.3.3. Test

- First reset the table so that it has 10 rows.

```sql
truncate table demo;
@insert
```

- Test-run the pipeline

```sql
begin
    dbms_cloud_pipeline.run_pipeline_once(
        pipeline_name => 'PIPELINE_EXPORT_DEMO'
    );
end;
/
```

- This may or may not export data. The reason is not clear. Proceed anyway.

- Reset the pipeline along with deleting table data
    - In this case, `purge_data => true` will delete the exported object files.

```sql
begin
    dbms_cloud_pipeline.reset_pipeline(
        pipeline_name => 'PIPELINE_EXPORT_DEMO',
        purge_data => true
    );
end;
/
```

<br/>

#### 2.3.4. Start the Pipeline

```sql
begin
    dbms_cloud_pipeline.start_pipeline(
        pipeline_name => 'PIPELINE_EXPORT_DEMO'
    );
end;
/
```

- This time 10 rows are exported to `demo_export` folder from the table.
    - An example of (uniquely generated) object file name: `DBMS_CLOUD_PIPELINE$8_1_20230814T095832584924Z.csv`

<br/>

#### 2.3.5. Final Test: New Data Inserted into the Table!

- Insert 10 more rows into demo table
    - This will lead to another export of 10 rows. The data pipeline works!

```sql
@insert
```

<br/>

#### 2.3.6. Clean-up: Stop and Drop the Pipeline

```sql
begin
    execute dbms_cloud_pipeline.stop_pipeline(pipeline_name => 'PIPELINE_EXPORT_DEMO')
    execute dbms_cloud_pipeline.drop_pipeline(pipeline_name => 'PIPELINE_EXPORT_DEMO')
end;
/
```
