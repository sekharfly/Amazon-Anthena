
# Athena 101

Athena is based on [PrestoDB](https://prestodb.io/) which is a Facebook-created open source project. Athena also supports [Hive DDL](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL), ANSI SQL and works with commonly used formats like JSON, CSV, Parquet etc. The idea behind Athena is that it is server less from an end-user perspective. Similar to Lambda, you only pay for the queries you run and the storage costs of S3.

In terms of AWS ecosystem, it seems to fit in a the use case of ad-hoc querying and simplified management. Looking at other products, Redshift provides a data store for complex, multiple-joins based business intelligence workloads and EMR provides a method to run highly distributed processing frameworks such as Hadoop and Spark. Athena fits in between. It does not require cluster management but is probably not as powerful.

## Terminology


 - **Tables** - Tables are essentially metadata that describes your data similar to traditional database tables. One important difference is that there is no relational construct.


 - **Databases** - Alogical grouping of tables. Also know as catalog or a namespace.
 
 - **SerDe** - Serializer/Deserializer, which are libraries that tell Hive how to interpret data formats. Athena uses SerDes to interpret the data read from Amazon S3. The following SerDes are supported
     - Apache Web Logs: "org.apache.hadoop.hive.serde2.RegexSerDe"
     - CSV: "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"
     - TSV: "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"
     - Custom Delimiters: "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"
     - Parquet: "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
     - Orc: "org.apache.hadoop.hive.ql.io.orc.OrcSerde"
     - JSON: “org.apache.hive.hcatalog.data.JsonSerDe” OR org.openx.data.jsonserde.JsonSerD



## Key Points

 - It uses an internal data catalog to store information and schemas about the databases and tables that you create for your data stored in Amazon S3. You can modify the catalog using the Hive data definition language (DDL) or directly via the console.
 
 - There is no data loading or transformation required. You can delete table definitions and schema without impacting the underlying data stored on Amazon S3. 
 
 - Not all Presto functions are supported. 
 
 - When you create, update, or delete tables, those operations are guaranteed ACID-compliant. For example, if multiple users or clients attempt to create or alter an existing table at the same time, only one will be successful.
 
 - No support for stored procedures

 - You can use Athena to query underlying Amazon S3 bucket data that's in a different region from the region where you initiate the query. This is useful because Athena, as of this article is only available in some regions
 
- Athena table names are case-insensitive. But if you are interacting with Apache Spark, then your table column names must be lowercase. Athena is case-insensitive but Spark requires lowercase table names.

- Amazon Athena is priced per query and charges based on the amount of data scanned by the query. You can store data in a variety of formats on Amazon S3. If you compress your data, partition, or convert it to columnar storage formats, you pay less because you scan less data. Converting data to the columnar format allows Athena to read only the columns it needs to process the query. 


## Use cases

 - Use for querying of any data in S3. If latency is not critical and queries can be run in the background this fits well. For e.g analysing ELB log data in S3 
 
 - Go server less across the stack. For example,  API gateway can be used to accept requests which are handled by Lambda which in turn can leverage Athena for queries. The only persistent service used will be S3.  Any static content generated from this can be delivered using S3's static website functionality.
 
 -  As with most things, you can mix and match AWS services based on your workloads. One possibility is to use an on-demand EMR cluster to process data and dump results to S3. Then use Athena to create adhoc tables and run reports. 
 
 - “Facebook uses Presto for interactive queries against several internal data stores, including their 300PB data warehouse. Over 1,000 Facebook employees use Presto daily to run more than 30,000 queries that in total scan over a petabyte each per day”
 Source : [https://prestodb.io/]()


## Limitations
 
 - There is no support for transactions. This includes any transactions found in Hive or Presto.
  
 - Athena table names cannot contain special characters, other than underscore (_).
 
 - There are also service limits, some of which can be increased by raising it with AWS
     - You can only submit one query at a time and you can only have 5 (five) concurrent queries at one time per account.
     - Query timeout: 30 minutes
     - Number of databases: 100
     - Table: 100 per database
     - Number of partitions: 20k per table
     - Amazon S3 buckets are limited to 100 and Athena also needs a separate bucket to log results.


# Getting Started

## Using Console

You can use the console to click through the web form to create databases and tables. This part is fairly intuitive. In this example, let's look at doing the same with query editor.

 - Download the CSV sample file or clone the git repository
 
 ```
 wget https://github.com/srirajan/athena/blob/master/sample_data/company_funding.csv
 
 ```
 or
 
 ```
 git clone https://github.com/srirajan/athena
 ```
 
 - Load the data files on your S3 bucket. You can create a single S3 bucket and sub folders under it. You will need the S3 URL in the examples below.
 
 - Now, login to the AWS console and go the Athena section. 
 
 - Then in the Query editor, create the database using the following.
 
 ```
 CREATE DATABASE sampledata;
 ```
 
 - Create the table. Note, you don't need to specify the file name. It automatically picks the CSV files under that folder.  The same logic applies if you have partitioned the data into sub folders. If you have partitioned the data, then you need to create table with the partition information as well.
 
 ```
CREATE EXTERNAL TABLE IF NOT EXISTS sampledata.companyfunding (
 permalink	STRING,
 company STRING,
 numEmps INT,
 category STRING,
 city STRING,
 state STRING,
 fundedDate STRING,
 raisedAmt	INT,
 raisedCurrency STRING,
 round STRING
 )
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = ',',
  'field.delim' = ','
) 
LOCATION 's3://aws-athena-data-jfj28fj3lt05kg84kkdj444/company_funding/';
 ```

 - Run a query
 
  ```
 SELECT * from sampledata.companyfunding LIMIT 20;
  ```
 
  - Now because this is just a view of S3 data, you can also create another table with limited columns.
  
 ```
CREATE EXTERNAL TABLE IF NOT EXISTS sampledata.companyfundingsmall (
 company STRING,
 raisedAmt	INT
 )
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = ',',
  'field.delim' = ','
) 
LOCATION 's3://aws-athena-data-jfj28fj3lt05kg84kkdj444/company_funding/';
 ```

    
- Run a query to find top 5 funded companies

```
SELECT * from sampledata.companyfundingsmall ORDER BY raisedAmt DESC LIMIT 5;
```
  

- Athena can also be used to read Cloudtrail data using the *com.amazon.emr.hive.serde.CloudTrailSerde*


```
CREATE EXTERNAL TABLE sampledata.cloudtrail (
eventversion STRING,
userIdentity STRUCT<
  type:STRING,
  principalid:STRING,
  arn:STRING,
  accountid:STRING,
  invokedby:STRING,
  accesskeyid:STRING,
  userName:STRING,
  sessioncontext:STRUCT<
    attributes:STRUCT<
      mfaauthenticated:STRING,
      creationdate:STRING>,
    sessionIssuer:STRUCT<
      type:STRING,
      principalId:STRING,
      arn:STRING,
      accountId:STRING,
      userName:STRING>>>,
eventTime STRING,
eventSource STRING,
eventName STRING,
awsRegion STRING,
sourceIpAddress STRING,
userAgent STRING,
errorCode STRING,
errorMessage STRING,
requestParameters STRING,
responseElements STRING,
additionalEventData STRING,
requestId STRING,
eventId STRING,
resources ARRAY<STRUCT<
  ARN:STRING,accountId:
  STRING,type:STRING>>,
eventType STRING,
apiVersion STRING,
readOnly STRING,
recipientAccountId STRING,
serviceEventDetails STRING,
sharedEventID STRING,
vpcEndpointId STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://2737690b14804c5ea383c41c83941d5f-logs/AWSLogs/759133634148/'
```

 -  Query top IAM users and activities by eventname 
```
select count (*) as events, useridentity.username, eventname from cloudtrail
where eventtime >= '2017-01-01T00:00:00Z' 
and useridentity.type = 'IAMUser'
group by useridentity.username, eventname
order by events desc;
```

 - Find the top eventsources

```
select count (*) as events, eventsource from sampledata.cloudtrail group by eventsource order by events desc;
```


- Reading large volumes of data can be further optimised by partitioning the data and using a columnar storage. This would optimise both the cost of running queries and the query time as well. Formats like ORC and Parquet are better suited for this as well. Here's another example provided by AWS that uses partitions. 
 
      
```
  CREATE EXTERNAL TABLE sampledata.flight_delays_csv (
    yr INT,
    quarter INT,
    month INT,
    dayofmonth INT,
    dayofweek INT,
    flightdate STRING,
    uniquecarrier STRING,
    airlineid INT,
    carrier STRING,
    tailnum STRING,
    flightnum STRING,
    originairportid INT,
    originairportseqid INT,
    origincitymarketid INT,
    origin STRING,
    origincityname STRING,
    originstate STRING,
    originstatefips STRING,
    originstatename STRING,
    originwac INT,
    destairportid INT,
    destairportseqid INT,
    destcitymarketid INT,
    dest STRING,
    destcityname STRING,
    deststate STRING,
    deststatefips STRING,
    deststatename STRING,
    destwac INT,
    crsdeptime STRING,
    deptime STRING,
    depdelay INT,
    depdelayminutes INT,
    depdel15 INT,
    departuredelaygroups INT,
    deptimeblk STRING,
    taxiout INT,
    wheelsoff STRING,
    wheelson STRING,
    taxiin INT,
    crsarrtime INT,
    arrtime STRING,
    arrdelay INT,
    arrdelayminutes INT,
    arrdel15 INT,
    arrivaldelaygroups INT,
    arrtimeblk STRING,
    cancelled INT,
    cancellationcode STRING,
    diverted INT,
    crselapsedtime INT,
    actualelapsedtime INT,
    airtime INT,
    flights INT,
    distance INT,
    distancegroup INT,
    carrierdelay INT,
    weatherdelay INT,
    nasdelay INT,
    securitydelay INT,
    lateaircraftdelay INT,
    firstdeptime STRING,
    totaladdgtime INT,
    longestaddgtime INT,
    divairportlandings INT,
    divreacheddest INT,
    divactualelapsedtime INT,
    divarrdelay INT,
    divdistance INT,
    div1airport STRING,
    div1airportid INT,
    div1airportseqid INT,
    div1wheelson STRING,
    div1totalgtime INT,
    div1longestgtime INT,
    div1wheelsoff STRING,
    div1tailnum STRING,
    div2airport STRING,
    div2airportid INT,
    div2airportseqid INT,
    div2wheelson STRING,
    div2totalgtime INT,
    div2longestgtime INT,
    div2wheelsoff STRING,
    div2tailnum STRING,
    div3airport STRING,
    div3airportid INT,
    div3airportseqid INT,
    div3wheelson STRING,
    div3totalgtime INT,
    div3longestgtime INT,
    div3wheelsoff STRING,
    div3tailnum STRING,
    div4airport STRING,
    div4airportid INT,
    div4airportseqid INT,
    div4wheelson STRING,
    div4totalgtime INT,
    div4longestgtime INT,
    div4wheelsoff STRING,
    div4tailnum STRING,
    div5airport STRING,
    div5airportid INT,
    div5airportseqid INT,
    div5wheelson STRING,
    div5totalgtime INT,
    div5longestgtime INT,
    div5wheelsoff STRING,
    div5tailnum STRING
)
    PARTITIONED BY (year STRING)
    ROW FORMAT DELIMITED
      FIELDS TERMINATED BY ','
      ESCAPED BY '\\'
      LINES TERMINATED BY '\n'
    LOCATION 's3://athena-examples/flight/csv/';
```
 
- Repair the table to add the partitions to the metadata
 
```
MSCK REPAIR TABLE sampledata.flight_delays_csv;
```
  
 - List partitions
  
```
SHOW PARTITIONS sampledata.flight_delays_csv;
```
  
  -  Query for Top 10 routes delayed by more than 1 hour. This query might take up to 60 seconds to run. Note queries in the console can run in the background and so you try other things. 
 
```
  SELECT origin, dest, count(*) as delays
  FROM sampledata.flight_delays_csv
  WHERE depdelayminutes > 60
  GROUP BY origin, dest
  ORDER BY 3 DESC
  LIMIT 10;
```


## JDBC

As of this article, Athena only supports Console based queries and a Java SDK. There is no direct API integration either. Here's an example using Java

 - Download AthenaJDBC41-1.0.0.jar & aws-java-sdk-1.11.77.jar from the git repo
 
 - Download the  AthenaJDBCTest01.java from the git repo
 
 - Replace s3_staging_dir with a s3 bucket in your account. This is the S3 location to which your query output is written. The JDBC driver then asks Athena to read the results and provide rows of data back to the user.
 
 - Create a file called athena_creds that contains the AWS Access key and AWS secret key
 
 - Update the database name in code
 
 - Compile using the jar files. For windows the path separator is ';'
 
 ```
 javac -cp ".:AthenaJDBC41-1.0.0.jar:aws-java-sdk-1.11.77.jar"  AthenaJDBCTest01.java 
 
 ```
 
 - Execute the class and it should list all tables in the database specified
  
  ```
  java -cp  ".:AthenaJDBC41-1.0.0.jar:aws-java-sdk-1.11.77.jar"  AthenaJDBCTest01
  ```


# References & further reading

 - [http://docs.aws.amazon.com/athena/latest/ug/what-is.html]()

 - [https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL]()
 
 - http://tech.marksblogg.com/billion-nyc-taxi-rides-aws-athena.html

 - https://aws.amazon.com/blogs/big-data/analyzing-vpc-flow-logs-with-amazon-kinesis-firehose-amazon-athena-and-amazon-quicksight/

 - https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/

 - https://aws.amazon.com/blogs/big-data/analyzing-data-in-s3-using-amazon-athena/
 

============




