Setting Up Weaviate as a Container in SPCS: A Step-by-Step Guide
1. Initial Setup and Configuration
* Log into Snowflake using SnowSQL:
snowsql -a "YOURINSTANCE" -u "YOURUSER"


* Setup User/Role:
USE ROLE ACCOUNTADMIN;
CREATE SECURITY INTEGRATION SNOWSERVICES_INGRESS_OAUTH
 TYPE=oauth
 OAUTH_CLIENT=snowservices_ingress
 ENABLED=true;
CREATE ROLE WEAVIATE_ROLE;
CREATE USER weaviate_user PASSWORD='weaviate123' DEFAULT_ROLE = WEAVIATE_ROLE DEFAULT_SECONDARY_ROLES = ('ALL') MUST_CHANGE_PASSWORD = FALSE;
GRANT ROLE WEAVIATE_ROLE TO USER weaviate_user;
ALTER USER weaviate_user SET DEFAULT_ROLE = WEAVIATE_ROLE;


* Create Database and Warehouse:
USE ROLE SYSADMIN;
CREATE OR REPLACE WAREHOUSE WEAVIATE_WAREHOUSE WITH
 WAREHOUSE_SIZE='X-SMALL'
 AUTO_SUSPEND = 180
 AUTO_RESUME = true
 INITIALLY_SUSPENDED=false;
CREATE DATABASE IF NOT EXISTS WEAVIATE_DB_001;
USE DATABASE WEAVIATE_DB_001;
CREATE IMAGE REPOSITORY WEAVIATE_DB_001.PUBLIC.WEAVIATE_REPO;


2. Compute Pools Setup
* Create Compute Pools:
USE ROLE SYSADMIN;
CREATE COMPUTE POOL IF NOT EXISTS JUPYTER_CP
MIN_NODES = 1
MAX_NODES = 1
INSTANCE_FAMILY = STANDARD_2
AUTO_RESUME = true;
CREATE COMPUTE POOL IF NOT EXISTS WEAVIATE_COMPUTE_POOL
 MIN_NODES = 1
 MAX_NODES = 1
 INSTANCE_FAMILY = STANDARD_2
 AUTO_RESUME = true;
CREATE COMPUTE POOL IF NOT EXISTS TEXT2VEC_POOL
 MIN_NODES = 1
 MAX_NODES = 1
 INSTANCE_FAMILY = STANDARD_2
 AUTO_RESUME = true;


3. Setup Files and Stages
* Create Stages for YAML and Data:
USE DATABASE WEAVIATE_DB_001;
CREATE OR REPLACE STAGE YAML_STAGE;
CREATE OR REPLACE STAGE DATA ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
CREATE OR REPLACE STAGE FILES ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
* Upload Configuration Files:
PUT file:///path/to/spec-j.yaml @yaml_stage;
PUT file:///path/to/spec-t2vec.yaml @yaml_stage;
PUT file:///path/to/t2vec.yaml @yaml_stage;


4. Building and Creating Docker Images
* Build Weaviate Docker Image:
docker build --rm --platform linux/amd64 -t weaviate -f weaviate.Dockerfile .
* Tag and Push the Docker Image to the Repository:
docker tag weaviate [repository_url]
docker push [repository_url]

5. Loading Data and Running Services
* Load Data into Snowflake Stage:
PUT file:///path/to/SampleJSON.json @DATA;


* Batch Process to Import Data into Weaviate:
# Python snippet to load and batch process data
with open("SampleJSON.json") as file:
 data = json.load(file)


with client.batch(batch_size=100) as batch:
 for record in data:
 batch.add_data_object(
 {
 "answer": record["Answer"],
 "question": record["Question"],
 "category": record["Category"]
 },
 "Question"
 )


6. Suspending and Resuming Services
* Suspend Services:
alter service weaviate suspend;
alter service TEXT2VEC suspend;
alter service jupyter suspend;


* Resume Services:
alter service weaviate resume;
alter service TEXT2VEC resume;
alter service jupyter resume;

* 7. Cleanup and Removal
* Remove Services and Resources:
DROP SERVICE WEAVIATE;
DROP SERVICE JUPYTER;
DROP SERVICE TEXT2VEC;
DROP STAGE DATA;
DROP STAGE FILES;
DROP IMAGE REPOSITORY WEAVIATE_DB_001.PUBLIC.WEAVIATE_REPO;
DROP DATABASE WEAVIATE_DB_001;
DROP WAREHOUSE WEAVIATE_WAREHOUSE;
DROP COMPUTE POOL WEAVIATE_COMPUTE_POOL;
DROP COMPUTE POOL JUPYTER_CP;
DROP COMPUTE POOL TEXT2VEC_POOL;
DROP USER weaviate_user;
DROP ROLE WEAVIATE_ROLE;
DROP SECURITY INTEGRATION SNOWSERVICES_INGRESS_OAUTH;