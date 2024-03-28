# alluxio-spark-trino-minio

### Employ the Alluxio cache file system with Spark, Trino, and Hive Metastore and with Minio as the S3 object store

---

## INTRO

This git repo provides a complete environment for demonstrating how to configure Alluxio's caching file system capability for use with SparkSQL and Trino as the query engines and Hive as the metastore.

This repo deploys Trino, Alluxio, Hive metastore, Spark and Minio. It configures a Trino Hive catalog to use the Alluxio caching engine. For the Community Edition of Alluxio, see: 

    https://docs.alluxio.io/os/user/stable/en/Overview.html

For the Enterprise Edition of Alluxio, see:

    https://docs.alluxio.io/ee-da/user/stable/en/Overview.html

## USAGE

### Step 1. Install Docker desktop 

Install Docker desktop on your laptop, including the docker-compose command.

    See: https://www.docker.com/products/docker-desktop/

### Step 2. Clone this repo

Use the git command to clone this repo (or download the zip file from the github.com site).

    git clone https://github.com/gregpalmr/alluxio-spark-trino-minio

    cd alluxio-spark-trino-minio

### Step 3. Launch the docker containers.

Launch the containers defined in the docker-compose.yml file using the command:

    docker-compose up -d

The command will create the network object and the docker volumes, then it will take some time to pull the various docker images. When it is complete, you see this output:

    $ docker-compose up -d
    Creating network "alluxio-spark-trino-minio_custom"
    Creating volume "alluxio-spark-trino-minio_trino-data" with local driver
    Creating volume "alluxio-spark-trino-minio_alluxio-data" with local driver
    Creating volume "alluxio-spark-trino-minio_minio-data" with local driver
    Creating volume "alluxio-spark-trino-minio_hive-metastore-data" with local driver
    Creating alluxio-master       ... done
    Creating alluxio-worker1      ... done
    Creating trino-coordinator    ... done
    Creating spark-master         ... done
    Creating spark-worker         ... done
    Creating minio                ... done
    Creating minio-create-buckets ... done
    Creating alluxio-mount-minio-bucket ... done
    Creating hive-metastore             ... done

### Step 4. View the buckets in Minio

Use the Minio web console to view the pre-staged bucket in Minio. Follow these steps:

- Point your web browser to http://localhost:9001
- Log in using the user id "minio" and the password "minio123"
- Click on the "Object Browser" link from the left margin menu.
- Click on the bucket link named "minio-bucket-1" and view the folder named "user" in the bucket
- Click on the folder named "user" view the folder named "hive"
- Click on the folder named hive and view the folder named "warehouse"

### Step 5. View the buckets in the Alluxio command line interface

If you want to use the Alluxio command line to view the bucket mount, you can follow these steps:

Launch a shell into the alluxio-master docker container:

    docker exec -it alluxio-master bash

Run the Alluxio file system command to view all mount points:

    alluxio fs mount

This will show you the Minio S3 bucket mounted within Alluxio:

    $ alluxio fs mount
    s3a://minio-bucket-1/user on  /user  (s3, capacity=-1B, used=-1B, not read-only, not shared, properties={alluxio.underfs.s3.inherit.acl=false, alluxio.underfs.s3.disable.dns.buckets=true, alluxio.underfs.s3.endpoint=http://minio:9000, s3a.accessKeyId=******, s3a.secretKey=******})
    /opt/alluxio/underFSStorage  on  /  (local, capacity=1006.85GB, used=8.29GB(0%), not read-only, not shared, properties={})

Then, you can list the directory tree with the command:

    alluxio fs ls -R /

### Step 6. View the Alluxio Web console

Point your web browser to the Alluxio Web console to view the Alluxio masters, workers and cache information.

     http://localhost:19999

### Step 7. View the Trino Web console

Point your web browser to the Trino Web console to view query status and history

     http://localhost:8080

The user id is "trino" and there is no password.

### Step 8. Test Trino with the Alluxio virtual file system

When using Trino with Alluxio Community Edition, you can specify Alluxio as the storage location when you define the table.  With the Enterprise Edition of Alluxio, you can use the Transparent URI feature which will automatically redirect references to s3 and s3a URIs to the native Alluxio URI (alluxio://).  But with this Community Edition of Alluxio, you must specify the alluxio:// URI within the table LOCATION specification.

Launch a bash session in the Trino coordinator container and run a CREATE TABLE command to create a table using the "minio" Trino cagtalog setup and the "s3a" URI. Then query the data. Use these commands:

    docker exec -it trino-coordinator bash

Then launch a Trino command line session:

    trino --catalog hive --debug

Then create a new table and copy data from an existing Trino schema and table using the commands:

    USE default;

    CREATE TABLE default.customer
          WITH (
            format = 'ORC',
            external_location = 'alluxio:///user/hive/warehouse/customer/'
          ) 
          AS SELECT * FROM tpch.tiny.customer;
 
 Then run a query against the new table with the Trino command:

    SELECT * FROM default.customer 
             WHERE acctbal > 3500.00 AND acctbal < 9000.00 
             ORDER BY acctbal LIMIT 25;
 
### Step 9. View the Alluxio cache storage usage

When Trino queries data using Alluxio's virtual file system, Alluxio will cache data to cache storage when it is first read from the under file system (in this case, Minio).

To see if the customer table's data files are being cached, go back to the bash session on the alluxio-master container and list the data files. You should see that some data is cached for the customer data set by the fact that the 8th column shows a percentage being cached. If it shows 100%, then Alluxio cached all of the data in that file.

    docker exec -it alluxio-master bash

Then run the Alluxio list command:

    alluxio fs ls -R /user/hive/warehouse/customer

This will show the following results which shows "100%" being cached in Alluxio cache storage.

    $ alluxio fs ls -R /user/hive/warehouse/customer
    -rw-r--r--  trino 80604 PERSISTED 03-27-2024 21:59:09:528 100% /user/hive/warehouse/customer/20240327_215903_00001_z3xv8_a9704c55-d8bb-4ffa-8a21-0b5f67841ca8

You can also view the Alluxio Web console you launched in Step 5 to see if any data is being cached by Alluxio (on the "Overview" page), or run the following Alluxio CLI command:

    alluxio fsadmin report

This will show that some worker node cache storage is being used:

    $ alluxio fsadmin report
    Alluxio cluster summary:
        Master Address: alluxio-master:19998
        ...
        Total Capacity: 1024.00MB
            Tier: MEM  Size: 1024.00MB
        Used Capacity: 78.71KB
            Tier: MEM  Size: 78.71KB
        Free Capacity: 1023.92MB

### Step 10. Test Spark with the Alluxio virtual file system

The Alluxio virtual file system will handle read/write operations from Spark jobs when they reference the alluxio:// filesystem scheme. If Spark is using the Hive Metastore, the Hive table definitions with the "external_location=alluxio://<directory_name>/" will be directed to Alluxio and Alluxio will read from its cache, if the file is already cached, or it will read from the under file system or UFS (Minio in this casE).

Launch a bash session in the Spark master container and run some Spark Scala commands to access the Hive table created in Step 9 above (Trino with Alluxio). And also access the data file directly without using the Hive metastore.  Start the bash session with the command:

    docker exec -it spark-master bash

Then start a Spark shell command line session:

    spark-shell 

The spark-shell command will load any defaults from the $SPARK_HOME/conf/spark-defaults.conf file which has been pre-staged with the Alluxio configuration.

Run the Spark Scala commands to query the Hive table:

       // Read the Hive table using the hive metastore
       // Note: It will read via the Alluxio virtual file system
	  
       import org.apache.spark.sql.hive.HiveContext

       val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)

       hiveContext.sql("USE default")

       hiveContext.sql("SHOW TABLES").show()

       hiveContext.sql("SELECT * FROM default.customer WHERE acctbal > 3500.00 AND acctbal < 9000.00 ORDER BY acctbal LIMIT 25").show()

       // Read the S3 bucket directly without using the Hive metastore
       // Note: It will read via the Alluxio virtual file system

       val df=spark.read.orc("alluxio:///user/hive/warehouse/customer/").show(25)

       // Create a new Hive table and write the new data via Alluxio
       val df = Seq((1, "John"), (2, "Jane"), (3, "Bob")).toDF("id", "name")

       df.write.option("path","alluxio:///user/hive/warehouse/my_spark_table").format("parquet").saveAsTable("default.my_spark_table");

       spark.catalog.listTables.show(false)

       spark.sql("SELECT * FROM default.my_spark_table LIMIT 10").show() 

To view the data files created by the Spark df.write.saveAsTable() operation, go back to the Alluxio web console at:

     http://localhost:19999

And click on the "Browse" tab at the top, then click on the "hive" folder and then the "warehouse" folder. The "test_table" folder will show the parquet files created for the new table in the my_spark_table directory. Notice that Alluxio is reporting that 100% of the data has been cached. That is because Alluxio has been configured to cache on write as well as cache on read.

### Step 11. Destroy the containers

If you want to destroy the containers but maintain the persistent volumes (and the Hive tables and Minio buckets), you can use the command:

     docker-compose down

If you don't intend to continue using the container volumes (and the Hive tables and Minio buckets), destroy the docker containers and clean up the docker volumes using the command:

     docker-compose down --volumes

---

Please direct questions or comments to gregpalmr@gmail.com
