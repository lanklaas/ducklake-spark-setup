# ducklake-spark-setup

Getting the duckdb jdbc driver info is at the bottom.

* Spin up a local postgres
 * Create the metadata db:
```sql
create database ducklake_catalog
```
* Spin up minio (or use some kind of s3)
* Create a test table with the duckdb cli:
```sql
SET s3_endpoint         = '192.168.1.180:9000'; --IP where minio is running
SET s3_access_key_id    = 'minio-user';
SET s3_secret_access_key= 'minio-password';
SET s3_region           = 'us-east-1';    -- any valid region
SET s3_url_style        = 'path';         -- or 'virtual'
SET s3_use_ssl          = 'false';    

ATTACH 'ducklake:postgres:dbname=ducklake_catalog host=localhost password=test user=postgres' AS my_ducklake (DATA_PATH 's3://ducklake');

USE my_ducklake;
create table alb as select * from 'local/path/to/any/file/alb.parquet';
```

* Run a local spark with the duckdb jdbc:
```bash
./bin/spark-sql \  
--jars  /home/me/git/hub/duckdb/duckdb-java/build/release/duckdb_jdbc.jar
```
* Create a spark table using the ducklake table:

```sql
create table if not exists test using jdbc options (dbtable  "(select * from alb) obj",driver "org.duckdb.DuckDBDriver", url "jdbc:duckdb:ducklake:postgres:postgresql://postgres:test@127.0.0.1:5432/ducklake_catalog", s3_endpoint "192.168.1.180:9000", s3_access_key_id "minio-user", s3_secret_access_key "minio-password", s3_region "us-east-1", s3_url_style "path", s3_use_ssl "false");
```
Maven has version `1.3.1.0` which might also work, but I did not test this process with that version and I saw a couple of ducklake related fixes in `1.3.2.0`

To build the jdbc driver, clone the repo: https://github.com/duckdb/duckdb-java, checkout tag `v1.3.2.0` and put the below dockerfile in the project root. Then run the docker build:

```dockerfile
# This version of debian gets an older version of libc (2.31)
# that is more compatible with older and newer code
FROM openjdk:17-jdk-bullseye

RUN apt update
RUN apt install -y build-essential
RUN apt install -y cmake

WORKDIR /source
COPY . .

RUN make release
```

```bash
docker build -t duckjav-v1.3.2 .
```
The build takes a long time. When it is done run the image and copy out the jdbc driver:

```bash
docker  run  -it  duckjav-v1.3.2  bash
# In a new shell (Get the container id via docker ps)
docker  cp  5fd5bbbb7568:/source/build/release/duckdb_jdbc.jar /home/me/git/hub/duckdb/duckdb-java/build/release/
```
