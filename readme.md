# Configure a Weaviate Container in SPCS: A Step-by-Step Guide

## Initial Setup

The code in this guide configures a sample Snowpark Container Services (SPCS). You should change the database name, warehouse name, image repository name, and other example values to match your deployment.

### 1. Log into Snowflake

Use the [SnowSQL](https://docs.snowflake.com/en/user-guide/snowsql) client to connect to Snowflake.

```bash  
snowsql -a "YOURINSTANCE" -u "YOURUSER"
```

### 2. Setup a `user` and a `role`

To setup a user and role, edit the `PASSWORD` field and run this SQL code:

```sql
USE ROLE ACCOUNTADMIN;
CREATE SECURITY INTEGRATION SNOWSERVICES_INGRESS_OAUTH
  TYPE=oauth
  OAUTH_CLIENT=snowservices_ingress
  ENABLED=true;
CREATE ROLE WEAVIATE_ROLE;
CREATE USER weaviate_user
  PASSWORD='weaviate123'
  DEFAULT_ROLE = WEAVIATE_ROLE
  DEFAULT_SECONDARY_ROLES = ('ALL')
  MUST_CHANGE_PASSWORD = FALSE;
GRANT ROLE WEAVIATE_ROLE TO USER weaviate_user;
ALTER USER weaviate_user SET DEFAULT_ROLE = WEAVIATE_ROLE;
```

### 3. Create a database and a warehouse

Create a database and warehouse to use with Weaviate. Edit the database name and repository name, then run this code.

```sql
USE ROLE SYSADMIN;
CREATE OR REPLACE WAREHOUSE WEAVIATE_WAREHOUSE WITH
  WAREHOUSE_SIZE='X-SMALL'
  AUTO_SUSPEND = 180
  AUTO_RESUME = true
  INITIALLY_SUSPENDED=false;
CREATE DATABASE IF NOT EXISTS WEAVIATE_DB_001;
USE DATABASE WEAVIATE_DB_001;
CREATE IMAGE REPOSITORY WEAVIATE_DB_001.PUBLIC.WEAVIATE_REPO;
```

### 4. Setup compute pools

Create compute pools.

```sql
USE ROLE SYSADMIN;
CREATE COMPUTE POOL IF NOT EXISTS JUPYTER_CP
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = STANDARD_2
  AUTO_RESUME = true;
CREATE COMPUTE POOL IF NOT EXISTS WEAVIATE_CP
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = STANDARD_2
  AUTO_RESUME = true;
CREATE COMPUTE POOL IF NOT EXISTS TEXT2VEC_CP
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = STANDARD_2
  AUTO_RESUME = true;
```

To check if the compute pools are active, run `DESCRIBE COMPUTE POOL`.

```sql
DESCRIBE COMPUTE POOL JUPYTER_CP;
DESCRIBE COMPUTE POOL WEAVIATE_CP;
DESCRIBE COMPUTE POOL TEXT2VEC_CP;
```

The compute pools are ready for use when they reach the `ACTIVE` or `IDLE` state.

### 5. Setup files and stages

Create stages for YAML and Data.

```sql
USE DATABASE WEAVIATE_DB_001;
CREATE OR REPLACE STAGE YAML_STAGE;
CREATE OR REPLACE STAGE DATA ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
CREATE OR REPLACE STAGE FILES ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
```

Upload your configuration files for your containers. You can find the files in this repo.

```sql
PUT file:///path/to/spec-jupyter.yaml @yaml_stage;
PUT file:///path/to/spec-text2vec.yaml @yaml_stage;
PUT file:///path/to/spec-weaviate.yaml @yaml_stage;
```

### 6 Build the Docker images

Exit the `snowsql` client. Build the Weaviate Docker image in your local shell.

```bash
docker build --rm --platform linux/amd64 -t weaviate -f weaviate.Dockerfile .
```

Log in to the Docker repository.

```bash
docker login YOUR_ACCOUNT_NAME_.registry.snowflakecomputing.com/THE_REPO_YOU_CREATED_ABOVE  -u YOUR_SNOWFLAKE_USERNAME
```

Tag and push the Docker image to the repository you created earlier.

```bash
docker tag weaviate YOUR_REPOSITORY_URL
docker push YOUR_REPOSITORY_URL
```

The `docker push` command looks like this.

```bash
docker push x0000000000000-xx00000.registry.snowflakecomputing.com/weaviate_db_001/public/weaviate_repo/weaviate
```

### 7. Load your data

Log in to the `snowsql` client. Be sure you are in the correct warehouse and database. (You may need to change roles to switch warehouses.)

```sql
use WAREHOUSE WEAVIATE_WAREHOUSE;
use DATABASE WEAVIATE_DB_001;
```

Then, load your data into the Snowflake stage.

```sql
PUT file:///path/to/SampleJSON.json @DATA;
```

### 8. Create the services

Create a service for each component.

```sql

CREATE SERVICE IF NOT EXISTS TEXT2VEC 
  min_instances=1 
  max_instances=1 
  compute_pool=TEXT2VEC_CP 
  spec=@yaml_stage/spec-text2vec.yaml;

CREATE SERVICE IF NOT EXISTS WEAVIATE
  MIN_INSTANCES = 1
  MAX_INSTANCES = 1
  COMPUTE_POOL = WEAVIATE_CP
  SPEC = @yaml_stage/spec-weaviate.yaml;

CREATE SERVICE IF NOT EXISTS jupyter
  MIN_INSTANCES = 1
  MAX_INSTANCES = 1
  COMPUTE_POOL = JUPYTER_CP
  SPEC = @yaml_stage/spec-jupyter.yaml;

```  

## Suspend and resume services

To suspend and resume services, run the following code in to the `snowsql` client.

### Suspend services
```sql
alter service WEAVIATE suspend;
alter service TEXT2VEC suspend;
alter service JUPYTER suspend;
```

### Resume services:
```sql
alter service WEAVIATE resume;
alter service TEXT2VEC resume;
alter service JUPYTER resume;
```

## Cleanup and Removal

To remove the services, run the following code in to the `snowsql` client.

```sql
DROP SERVICE WEAVIATE;
DROP SERVICE JUPYTER;
DROP SERVICE TEXT2VEC;
DROP STAGE DATA;
DROP STAGE FILES;
DROP IMAGE REPOSITORY WEAVIATE_DB_001.PUBLIC.WEAVIATE_REPO;
DROP DATABASE WEAVIATE_DB_001;
DROP WAREHOUSE WEAVIATE_WAREHOUSE;
DROP COMPUTE POOL WEAVIATE_CP;
DROP COMPUTE POOL JUPYTER_CP;
DROP COMPUTE POOL TEXT2VEC_CP;
DROP USER weaviate_user;
DROP ROLE WEAVIATE_ROLE;
DROP SECURITY INTEGRATION SNOWSERVICES_INGRESS_OAUTH;
```