# DynamoDB-Direct-Data-Access-with-Hive
This Article provides step by step process of cloning AWS dynamoDB data using HDP Hive. When I was researching this for my use, no reference to use below outside EMR was available, Hence decided to share it for anyone who may benefit from this.

# Prerequisites:

	- Up and Running HDP Cluster available
	- Usable DynamoDB tables with secret and Access key of a read access user and is reachable from hadoop nodes.
	- maven installed locally to build few jars

# Steps to Configure DynamoDB Direct Access:

1) Download below jars and copy it to "/usr/hdp/2.6.x.x-yyy/hive/auxlib/" and /usr/hdp/2.6.x.x-yyy/hive2/auxlib/ on all Hive servers and edge nodes [make sure to use latest versions of below jars as time passes]

     http://central.maven.org/maven2/com/amazonaws/aws-java-sdk-dynamodb/1.11.445/aws-java-sdk-dynamodb-1.11.445.jar
     http://central.maven.org/maven2/com/amazonaws/aws-java-sdk-core/1.11.445/aws-java-sdk-core-1.11.445.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.9.7/jackson-databind-2.9.7.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2.9.7/jackson-annotations-2.9.7.jar
     http://central.maven.org/maven2/com/fasterxml/jackson/core/jackson-core/2.9.7/jackson-core-2.9.7.jar

2) Build below Repository from github and copy 2 jars  emr-dynamodb-hadoop-x.x.x.jar and emr-dynamodb-hive-x.x.x-SNAPSHOT-jar-with-dependencies.jar

      $ cd 
      $ git clone https://github.com/awslabs/emr-dynamodb-connector 
      $ cd  emr-dynamodb-connector/ 
      $ mvn clean install

3) In the Target directories of the build, above mentioned jars will be present, place them in hive directories /usr/hdp/2.6.x.x-yyy/hive2/auxlib/ and /usr/hdp/2.6.x.x-yyy/hive/auxlib/ .

4) Add below 4 configurations in custom hive-site.xml and tez-site.xml

      - fs.s3.awsSecretAccessKey=##SECRET_KEY##
      - fs.s3.awsAccessKeyId=##ACCESS_KEY##
      - dynamodb.awsSecretAccessKey=##SECRET_KEY##
      - dynamodb.awsAccessKeyId=##ACCESS_KEY##

5) Make sure to remove existing jackson*.jars from /usr/hdp/2.6.x.x/hive/lib/ to another folder when replacing it with jars downloaded in step 1

6) Restart Hive and any other services in HDP which requires the same. Steps till now will make sure you can access data in DynamoDB via hive.

# Steps to Clone DynamoDB data using Hive

1) Create and External Table in hive using below command via beeline

	- 	CREATE EXTERNAL TABLE dynamodb.UserStorage_201810 (userId string,Name string,age bigint) 
		ROW FORMAT SERDE 'org.apache.hadoop.hive.dynamodb.DynamoDBSerDe' STORED BY 'org.apache.hadoop.hive.dynamodb.DynamoDBStorageHandler'
		TBLPROPERTIES ("dynamodb.table.name"="UserInfo","dynamodb.region"="eu-west-2","dynamodb.throughput.read.percent"=".5000",
		"dynamodb.column.mapping"="userId:userId,Name:Name,age:age");
		
		dynamodb.table.name --> Exact Name of DynamoDB Table
		dynamodb.region --> Region name where table is located
		dynamodb.throughput.read.percent --> what percentage of DynamoDB table read capacity units can mapreduce job can consume to copy data : set a moderate % as required in above conf I have used 0.5000 indicating I can use 50% of read units [remember 1unit is 4kb per sec and aws cost will be through the roof if auto scaling is enabled and table contains huge amount of data]


