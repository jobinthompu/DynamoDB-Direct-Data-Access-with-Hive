# DynamoDB-Direct-Data-Access-with-Hive
This Article provides step by step process of cloning AWS dynamoDB data using HDP Hive. When I was researching this for my use, no reference to use below outside EMR was available, Hence decided to share it for anyone who may benefit from this.

## Prerequisites:
```
	1) Up and Running HDP Cluster available
	2) Usable DynamoDB tables with secret and Access key of a read access user and is reachable from hadoop nodes.
	3) maven installed locally to build few jars
```
## Steps to Configure DynamoDB Direct Access:

1) Download below jars and copy it to "/usr/hdp/2.6.x.x-yyy/hive/auxlib/" and /usr/hdp/2.6.x.x-yyy/hive2/auxlib/ on all Hive servers and edge/client nodes [make sure to use latest versions of below jars as time passes]
	```
     http://central.maven.org/maven2/com/amazonaws/aws-java-sdk-dynamodb/1.11.445/aws-java-sdk-dynamodb-1.11.445.jar
     http://central.maven.org/maven2/com/amazonaws/aws-java-sdk-core/1.11.445/aws-java-sdk-core-1.11.445.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.9.7/jackson-databind-2.9.7.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2.9.7/jackson-annotations-2.9.7.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-core/2.9.7/jackson-core-2.9.7.jar
	```
2) Build below Repository from github and copy 2 jars  emr-dynamodb-hadoop-x.x.x.jar and emr-dynamodb-hive-x.x.x-SNAPSHOT-jar-with-dependencies.jar
	```
    $ cd 
    $ git clone https://github.com/awslabs/emr-dynamodb-connector 
    $ cd  emr-dynamodb-connector/ 
    $ mvn clean install
	```
3) In the Target directories of the build, above mentioned jars will be present, place them in hive directories /usr/hdp/2.6.x.x-yyy/hive2/auxlib/ and /usr/hdp/2.6.x.x-yyy/hive/auxlib/ .

4) Add below 4 configurations in custom hive-site.xml and tez-site.xml
	```
    fs.s3.awsSecretAccessKey=##SECRET_KEY##
    fs.s3.awsAccessKeyId=##ACCESS_KEY##
    dynamodb.awsSecretAccessKey=##SECRET_KEY##
    dynamodb.awsAccessKeyId=##ACCESS_KEY##
	```
5) Make sure to remove existing jackson*.jars from /usr/hdp/2.6.x.x/hive/lib/ to another folder when replacing it with jars downloaded in step 1

6) Restart Hive and any other services in HDP which requires the same. Steps till now will make sure you can access data in DynamoDB via hive.

## Steps to Clone DynamoDB data using Hive

1) Create and External Table in hive using below command via beeline
	```
	CREATE EXTERNAL TABLE dynamodb.UserInfo (userId string,Name string,age bigint,lastModified bigint) 
	ROW FORMAT SERDE 'org.apache.hadoop.hive.dynamodb.DynamoDBSerDe' STORED BY 'org.apache.hadoop.hive.dynamodb.DynamoDBStorageHandler'
	TBLPROPERTIES ("dynamodb.table.name"="UserInfo","dynamodb.region"="eu-west-2","dynamodb.throughput.read.percent"=".5000",
	"dynamodb.column.mapping"="userId:userId,Name:Name,age:age,lastModified:lastModified");
	```
	
	Explanation of properties used Above
	```
	dynamodb.table.name --> Exact Name of DynamoDB Table
	dynamodb.region --> Region name where table is located
    dynamodb.throughput.read.percent --> what percentage of DynamoDB table read capacity units can mapreduce job can consume to copy data : set a moderate % as required in above conf I have used 0.5000 indicating I can use 50% of read units [remember 1unit is 4kb per sec and aws cost will be through the roof if auto scaling is enabled and table contains huge amount of data]
	```
	[More Details Here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EMRforDynamoDB.PerformanceTuning.Throughput.html)

2) Create an ORC ACID Internal table as I was looking at transactional data with updates in Dynamodb data, not just inserts.

	```
	CREATE TABLE dynamodb.UserInfo_orc (userid string,Name string,age bigint,lastModified bigint,lastModified_date timestamp) 
	CLUSTERED BY (userid) INTO 256 BUCKETS 
	stored as ORC TBLPROPERTIES ("immutable"="false","transactional"="true");
	```
3) [Initial Load] Insert data from external dynamoDB table into created orc table as below [Initial Load]:

	```
	insert into dynamodb.UserInfo_orc select userid,Name,age,lastModified,cast(from_unixtime(floor(CAST(lastModified AS BIGINT)/1000), 'yyyy-MM-dd HH:mm:ss.SSS') as timestamp) from dynamodb.vw_UserDevices;
	```
4) [Incremental Load] Create a table from the external Table with only to the latest data in external table, run every hour [I capture data 1 hour before the latest data in my ORC table and replace it every hour, creating another table helps with not having table locks while insert from external table happens which may take long time due to dynamoDB read limitation]
	
	```
	select max(lastmodified)-3600000 from dynamodb.UserInfo_orc;  --> ${lastupdateddate}
	Drop Table dynamodb.UserInfo_Temp;
	Create Table dynamodb.UserInfo_Temp stored as orc as select * from dynamodb.UserInfo where lastModified>=${lastupdateddate};
	```
Note: This requires indexes in dynamodb created properly using lastModified esle it may read entire table to get incremental data

5) Once the Temperory table with latest records are loaded, as the ORC table is created as ACID table, I delete the userid(s) present in temperory table from ORC table, which are updated in this case. in My case userid is the PK in Dynamodb UserInfo table 

	```
	delete from dynamodb.UserInfo_orc where userid in (select userid from dynamodb.UserInfo_Temp); 
	```
6) Once Delete is done, Its time to insert incremental data into ORC Table:

	```
	insert into dynamodb.UserInfo_orc select userid,Name,age,lastModified,cast(from_unixtime(floor(CAST(lastModified AS BIGINT)/1000), 'yyyy-MM-dd HH:mm:ss.SSS') as timestamp) from dynamodb.UserInfo_Temp;
	```
7) If you are running the incremental too often like I do, its a good idea to do compaction after insert to reduce the files

	```
	ALTER TABLE dynamodb.UserInfo_orc COMPACT 'MAJOR';
	```
Note: I use Apache NiFi to automate the end to end flow Along with monitoring on each step via slack, the job runs hourly.

## References:
	
* [ AWS Docuemntation ](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EMRforDynamoDB.html)
* [ EMR Github ](https://github.com/awslabs/emr-dynamodb-connector)