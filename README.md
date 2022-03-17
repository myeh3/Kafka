# Real Time Data Ingestion with Kafka and Dremio



> A hand by hand implementation steps with Docker, SAP C4C, Confluent Kafka,
> Azure Data Lake Storage, Dremio :hammer: 
> Guided By Michal Zawadzki, Data Platform Team Lead, Dyvenia :male-teacher:
> Guided By Diane Woodbride, Program Director, USF :female-teacher:
> Written By Jeff Yeh, Data Engineer Intern, Velux :male-student: 

This instruction focuses on the technical implementation of real-time data ingestion pipeline development with Kafka and Dremio under docker environment

## :memo: What are the data source and sink target?

In this deployment case, the data source is from SAP C4C OData V2 Service Request API, and the sink target is the Azure Data Lake Storage Gen2 and Dremio.

![](https://i.imgur.com/Cc20BzM.png)



##  :orange_book:Data Overview

There are 177 data fields returned by the [SAP C4C OData V2 Service Request API.](https://drive.google.com/file/d/1GTpN-7Dtw8LlcC5MusAY1uvbToUpJ1z8/view?usp=sharing)

### Potential Problems:

(1) **Redundant Data Fields**: several fields which are not useful for business but contains much volume, and we need to remove these fields during the transmission. For example, __metadata, ServiceRequestTextCollection, ServiceRequestPart, and so on. [The required fields are stored in the list](https://docs.google.com/spreadsheets/d/1YAL1HXzFLVBhNwWL8TEL-6vexZ7OcgwA/edit?usp=sharing&ouid=112368109893676306483&rtpof=true&sd=true).


(2) **Date Data Format**: the date related data fields are encode as Unix timestamp with /, (, and ). We will need to parse these data and transform them into timestamp

![](https://i.imgur.com/PlUN8Jd.png)


## Step 1: Install docker

[Download desktop docker](https://www.docker.com/products/personal) and sign up a docker ID

![](https://i.imgur.com/OImiWU6.png)

If you are a Windows user, you would need to set up WSL2. before running docker. WSL(Windows Subsystem for Linux) is a official tool to support user a Linux environment in their Windows system.


Open your terminal and input:

``` wsl --install ```

After the installation completes, we can run the docker with Ubuntu 20.04 environment which could be installed from Microsoft Store.

![](https://i.imgur.com/e7g51ek.png)

:rocket: 

## Step 2: Prepare Kafka docker Images

First, we need to download the yml file from confluent resource. The yml file is like a docker version schema which define the container names, docker images used for each containers, ports, and so on.

```curl -s -o docker-compose.yml https://raw.githubusercontent.com/confluentinc/cp-all-in-one/6.2.1-post/cp-all-in-one/docker-compose.yml```


:::info
:bulb: Most of the docker images used in the yml is fine to be found from the docker hub. However, for the container:connect, since we are going to use HTTP Source Connector specifically in this case, we need to update the connector image. Please see the instruction below.
:::

**(1)build a new docker image which contains the HTTP Source Connector related setting**

First, create a file named “Dockefile” and paste the command below

```Dockerfile
FROM confluentinc/cp-kafka-connect:6.2.0

RUN confluent-hub install --no-prompt castorm/kafka-connect-http:0.8.6

WORKDIR /
```

Then, use terminal for below command to build the docker images by the Dockerfile we created and name it as “http_adls_connect”

```docker build . --tag http_adls_connect```

**(2)changed the image of connect to the docker image we build.**

Now, let’s open our yml file and replace the image part of connect to the image name we just build “http_adls_connect”

![](https://i.imgur.com/CP2alqh.png)


:cake: **Congratulation! We have prepared all we need to initiate docker containers for Confluent Kafka.** 


Then, let’s initiate the docker containers by execute the command below. This command will grab the yml file in your current directory and initiate container based on the definition of yml file. It may take 5–10 mins to download the new docker images specified in your yml file if you didn’t install them before.

```docker-compose up -d```

If the process is successful, you should see the containers show as green in your docker desktop.

![](https://i.imgur.com/d2tpqLY.png)


## Step 3: Set up Kafka source connector, topic, and KSQL

There are many ways to implement this step. First, you could use confluent control center to configure the setting. Secondly, you could use docker cli of each container with docker desktop as well.

In this instruction, I will use the third option: docker exec command to set up docker container configuration directly from my local terminal.

### Create Topics
![](https://i.imgur.com/SbIbE18.png)

We need to create two topics in total. The first topic “service_request” is used to process the original data sourced from the Odata API. Let’s open the local terminal and input below command.

```docker exec broker kafka-topics --create --bootstrap-server localhost:9092 --topic service_request --replication-factor 1 --partitions 3```


The second topic “service_request_date_transformed_avro is used to process the KSQL date and avro transformation. Moreover, it would be the topic to sink our data to target data end.


```docker exec broker kafka-topics --create --bootstrap-server localhost:9092 --topic service_request_date_transformed_avro --replication-factor 1 --partitions 3```

### Create Schema Registry

For Schema Registry, it is easier to manage it through Confluent Control Center. Go to the http://localhost:9021/clusters. Then click the cluster your topics in.

![](https://i.imgur.com/vGwDUB8.png)

Click topic and choose the topic you would like to set up its schema registry.

![](https://i.imgur.com/Z4mVqfx.png)

Click Schema and input [the json schema we prepared](https://drive.google.com/file/d/1szlqk4SKYCaX0CNdMrcpy8qmkztVRKrR/view?usp=sharing), then click Create. 

![](https://i.imgur.com/t33h0Df.png)



### Create Source Connector

The source connector defines the data source(http request url), request method, request header, credentials, and other related parameters.

We can encapsulate our connector setting into a “Create_Source.txt” file and config it with docker exec.


```docker exec -i connect bash < Create_Source.txt```

This is the setting of Create_Source.txt,


```curl -X POST -H "Content-Type: application/json" --data '{"name": "c4c_connector","config": {"connector.class": "com.github.castorm.kafka.connect.http.HttpSourceConnector","tasks.max": 1,"http.offset.initial": "timestamp=2021-10-17T00:00:00.00Z","http.request.url": "https://my336539.crm.ondemand.com/sap/c4c/odata/v1/c4codataapi/ServiceRequestCollection","http.request.method": "GET","http.request.headers": "Content-Type: application/json","http.request.params": "$filter=EntityLastChangedOn ge datetimeoffset'"'"'''${offset.timestamp}'''"'"'&$format=json","http.auth.type": "Basic","http.auth.user": "enter-your-account","http.auth.password": "enter-your-password","http.response.list.pointer": "/d/results","http.response.record.offset.pointer": "key=/ObjectID, timestamp=/EntityLastChangedOn","http.response.record.mapper": "com.github.castorm.kafka.connect.http.record.StringKvSourceRecordMapper","http.response.record.timestamp.parser": "com.github.castorm.kafka.connect.http.response.timestamp.RegexTimestampParser","http.response.record.timestamp.parser.regex.delegate": "com.github.castorm.kafka.connect.http.response.timestamp.EpochMillisOrDelegateTimestampParser","http.response.record.timestamp.parser.regex": "(?:\/Date\\((.*?)\\)\/)","http.timer.interval.millis": 60000,"http.timer.catchup.interval.millis": 10000,"key.converter": "org.apache.kafka.connect.storage.StringConverter","value.converter": "org.apache.kafka.connect.storage.StringConverter","key.ignore": "true","value.converter.schemas.enable": "false","errors.retry.delay.max.ms": 60000,"errors.retry.timeout": 300000,"errors.tolerance": "all","errors.log.enable": true,"errors.log.include.messages": true,"kafka.topic": "service_request"}}' http://connect:8083/connectors -w "\n"```


### K-SQL

KSQL is a SQL based tool in Kafka ecosystem. It provides stream and table two different data processing ways for user to handle data from the topic.

For the “LastChangedOn” date data format problem we mentioned at the beginning, we could use a KSQL function:”FROM_UNIXTIME” to parse the date format to timestamp. Moreover, we should remove the redundant fields in the transformed Stream and Table too.

![](https://i.imgur.com/C6dGGU4.png)
![](https://i.imgur.com/Py2JF3T.png)

**(1)Create Stream**

KSQL Stream will listen to the specified topic for every records sent from the source connector.

```docker exec -i ksqldb-server ksql < Create_Stream.txt```
```docker exec -i ksqldb-server ksql < Create_transformed_Stream.txt```

Since the KSQL commands are too long, I provide two links below for download directly.
* [Create_Stream.txt](https://drive.google.com/file/d/1V5QNm4ui5KnHngk6hE1xUotcY3aH-Psh/view?usp=sharing)
* [Create_transformed_Stream.txt](https://drive.google.com/file/d/1dBw-GQwCbDhs-22drFxaUDrF5hhStgps/view?usp=sharing)

**(2)Create Table**

Compared to KSQL Stream, KSQL Table is like a relational database which upsert the latest records based on primary key(in this case, ObjectID is the primary key)we define.

```docker exec -i ksqldb-server ksql < Create_Stream.txt```
```docker exec -i ksqldb-server ksql < Create_transformed_Stream.txt```


* [Create_Table.txt](https://drive.google.com/file/d/1u-It-VzgSLvanxStn7Y_xGDP9mcPrQDD/view?usp=sharing)
* [Create_transformed_Table.txt](https://drive.google.com/file/d/1865OM4eN2u29eV_kawtfxP0FdzayRuV3/view?usp=sharing)

Now, we have used KSQL to transform our original data successfully, removed redundant fields, and solved the date format problem. Let’s now start to set up the sink connector to send our data to the target system.

![](https://i.imgur.com/yMSGkDy.png)
 *The EntityLastChangedOn original format* 

![The EntityLastChangedOn original format](https://i.imgur.com/3oFJlTj.png)
 *The EntityLastChangedOn format is transformed by KSQL* 


## Step4 : Sink data to PostgreSQL container (Optional)


This is an experiment session, you are free to skip if you prefer to use ADLS only. In this session, we would like to use JDBC sink connector to send our data to PostgreSQL relational database.

JDBC sink connector requires message transmitted with “schema” and “payload” [as the example](https://drive.google.com/file/d/1rXnE9tmHQCDGkLOMvcdVyrmRjBrPbjKd/view?usp=sharing). However, in our case, what we retrieve from the API is pure JSON. Thus, we need to use KSQL to transform our pure JSON data as Avro format in KSQL Stream(Avro format will bring the schema and value when we sink data to downstream).

### Install PostgreSQL Docker Container
We could use docker images to build PostgreSQL in docker. First, let’s modify our yml file to add 2 more containers: pg_container and pgadmin4_container.([you could use the yml file I modified earlier here](https://drive.google.com/file/d/19ciKPcugiXUCjtei2qu5X4DUubmVZ1op/view?usp=sharing)). The pg_container simulate the PostgreSQL environment and pgadmin4_container is a frontend tool to allow you monitor your PostgreSQL database.

```
db:
    container_name: pg_container
    image: postgres
    restart: always
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: root
      POSTGRES_DB: test_db
    ports:
      - "5432:5432"
pgadmin:
    container_name: pgadmin4_container
    image: dpage/pgadmin4
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
    ports:
      - "5050:80"

```


Then, please run docker-compose again in your terminal to trigger the activation of yml file.

```docker-compose up -d```


After that, please go to the localhost:5050 to access your pgadmin, and login with account: admin@admin.com and password:root.

![](https://i.imgur.com/v5zo3Cl.png)



Now, we need to link our pgadmin with our database host address. To get the ip, we need to use the command below in our local terminal. First, we need to use docker ps to get the container id of pg_container.

The 52dcc0a427f5 is the container id of pg_container got from docker ps

```docker ps```

![](https://i.imgur.com/bI5KSuH.png)


Then, let’s use below command to get the IPAddress of our pg_container server.

```docker inspect 52dcc0a427f5 | findstr IPAddress```

Once you got the IPAddress, go back to your pgadmin and create a new server to link to your IPAddress of pg_container.(default username : root, password : root)

![](https://i.imgur.com/QSyN34R.png)


### Install JDBC Connector
Open a terminal and use docker exec to install JDBC connector in our connect container.
```docker exec -i connect bash < confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:5.4.1
cd /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib
curl https://maven.xwiki.org/externals/com/oracle/jdbc/ojdbc8/12.2.0.1/ojdbc8-12.2.0.1.jar -o ojdbc8-12.2.0.1.jar
Create JDBC Sink Connector
Let’s use the docker exec to create a JDBC sink connector to sink data to our postgreSQL.
docker exec -i connect bash < Create_Sink_JDBC.txt
This is the setting of Create_Sink_JDBC.txt,
curl -X POST -H "Content-Type: application/json" --data '{"name": "c4c_JDBC_sink","config": {
 "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
   "connection.user": "root",
   "connection.password": "root",
   "connection.url": "jdbc:postgresql://pg_container:5432/postgres",
   "transforms":"flatten",
   "transforms.flatten.type":"org.apache.kafka.connect.transforms.Flatten$Value",
   "topics": "service_request_date_transformed_avro",
   "key.converter":"org.apache.kafka.connect.storage.StringConverter",
   "pk.mode":"record_value",
   "pk.fields":"OBJECTID",
   "insert.mode": "upsert",
   "auto.create": "true",
   "auto.evolve": "true"
}}' http://connect:8083/connectors -w "\n" `

```


Once the sink connector is created, the data will flow to your PostgreSQL container and the table name would be your topic name.


![](https://i.imgur.com/eNFEztW.png)



## Step 5: Sink data to Azure Data Lake Storage

Now, let’s sink our data to Azure Data Lake Storage.

### Install Microsoft Azure Storage Explorer
Azure Storage Explorer is a tool which allows you to monitor and manage your cloud storage on Azure.[ You could download it free](https://azure.microsoft.com/en-us/features/storage-explorer/#overview).

Once installed, you could connect your Azure Storage account with the "Storage account or service”, then choose account name and key to input your credentials.

![](https://i.imgur.com/S1BB0Kw.png)
![](https://i.imgur.com/R8nyCe8.png)
![](https://i.imgur.com/KxWCPaU.png)

### Create AzureGen2 Sink Connector

Let’s use the docker exec to create a AzureGen2 sink connector to sink data to our ADLS.


```docker exec -i connect bash < Create_Sink_ADLS.txt```

This is the setting of Create_Sink_ADLS.txt,

```
curl -X POST -H "Content-Type: application/json" --data '{"name": "c4c_ADLS_sink","config": {"ssl.client.auth": "none",
    "time.interval": "HOURLY",
    "input.data.format": "JSON",
    "output.data.format": "JSON",
    "max.retry.time.ms": "10000",
    "confluent.topic.sasl.mechanism": "PLAIN",
    "name": "azuresink",
    "connector.class": "io.confluent.connect.azure.datalake.gen2.AzureDataLakeGen2SinkConnector",
    "tasks.max": "1",
    "topics": "service_request_date_transformed_avro",
    "topics.dir": "*****",
    "format.class": "io.confluent.connect.azure.storage.format.json.JsonFormat",
    "flush.size": "3",
    "partitioner.class": "io.confluent.connect.storage.partitioner.HourlyPartitioner",
    "locale": "en_DK",
    "timezone": "Europe/Copenhagen",
    "confluent.license": "*******",
    "confluent.topic.bootstrap.servers": "broker:29092",
    "azure.datalake.gen2.account.name": "*****",
    "azure.datalake.gen2.access.key": "*******"}}' http://connect:8083/connectors -w "\n"
```

Once the sink connector is created, the data will flow to your ADLS and be partitioned by year -> month ->day -> hour

![](https://i.imgur.com/RhxGzLa.png)



## Step 6: Install Dremio and Viadot docker images

Dremio is a data lakehouse engine which could integrate multiple data source and build high efficiency BI solutions. Viadot is a python package which could be used to connect to our Dremio for data manipulation. Our expectation in this step is to build Dremio docker container and connect it to our Azure Data Lake Storage.

### Install Viadot
First, we need to get the data from the github repo,

```
git clone https://github.com/dyvenia/viadot.git && \
cd viadot/docker && \
./update.sh
```

Then, run the file,

```./run.sh```

This file will help your pull the Viadot docker images for later Dremio container deployment use. [You could see more introduction about Viadot from the page.](https://github.com/dyvenia/viadot/tree/dev#local-setup)

### Install Dremio

Let’s download the files we need from the [link](https://medium.com/@myeh3/real-time-data-ingestion-with-kafka-and-dremio-b2e04232fa9#:~:text=Dremio%20is%20a,an%20account%20first.). Then, go to the folder, and run

```docker-compose up -d```

Go to the localhost:9047 to view the Dremio. You will need to create an account first.


![](https://i.imgur.com/PcajQ7z.png)




## Step 7: Build data lakehouses in Dremio - WebUI

There are two ways to conduct the dremio setting:WebUI and HTTP request. In Step 7, let's focus on how to implement dremio with WebUI. In Step 8, we will deep dig into the HTTP request.

### Link to Source
After login, go the Data Lakes session, and create a new connection to your ADLS.

![](https://i.imgur.com/CVhxRQC.png)
![](https://i.imgur.com/fV9d8E6.png)

### Create Physical Dataset
Go to the folder in ADLS, and find out the dataset/folder(if you have multiple dataset to consume under it), then click the convert button and specify the data format.

After that execution, you will notice the dataset/folder turns to purple icon. We called this purple actual dataset in your data lakes “ Physical Dataset”.
![](https://i.imgur.com/CnePGu7.png)


### Create Virtual Dataset
The virtual dataset(green icon)is the query result based on the physical dataset. In our case, our physical dataset “kafka" contains all of the historical update of same documents. Thus, in virtual layer, we need to find out a solution to provide both historical data and the latest data for users.

First, let’s create a Space called “sandbox”,

![](https://i.imgur.com/IJj2pu9.png)

Then, we could click “ New Query “ to create new virtual dataset.

![](https://i.imgur.com/pqyqQuf.png)

The query is based on SQL scripts. In my implementation, I created two virtual dataset. One for all historical data demonstration, and the other one only shows the latest data with Window function setting(partition by ObjectID and order by LastChangedOn).

![](https://i.imgur.com/1olxLAl.png)

Below are the two SQL scripts I use for the two virtual dataset,

* [Dremio_Historical](https://drive.google.com/file/d/13-s7LxFWCzhiPaIf-WBQMDUUDd9awQ1I/view?usp=sharing)
* [Dremio_Latest](https://drive.google.com/file/d/1SWJSqu0luFm8IVClisGGb25Dz4EWsQzI/view?usp=sharing)

## Step 8: Build data lakehouses in Dremio - HTTP Request

The information of the dremio API is referred from the [dremio documentation page ](https://docs.dremio.com/software/rest-api/catalog/post-catalog/).

### Link to Source
To link to data source, we first need to get the authentication(token) with below command.
```
curl --location --request POST 'http://localhost:9047/apiv2/login' \
--header 'Content-Type: application/json' \
--data-raw '{
  "userName": "tan50513",
  "password": "50513TAn++"
}
'
```

We will receive something like this, remember the token we receive:

```
{
    "token": "jn5h2ds081d80a588j6doijueq",
    "userName": "tan50513",
    "firstName": "Jeff",
    "lastName": "Yeh",
    "expires": 1646709523530,
    "email": "tan50513@gmail.com",
    "userId": "f3383a1e-fc46-449f-80a7-0bedb1e0f549",
    "admin": true,
    "clusterId": "dc3b3447-8d83-4d9f-b9da-01b76ac7d02b",
    "clusterCreatedAt": 1644430453345,
    "version": "18.1.0-202109222258120166-963adceb",
    "permissions": {
        "canUploadProfiles": true,
        "canDownloadProfiles": true,
        "canEmailForSupport": true,
        "canChatForSupport": false
    },
    "userCreatedAt": 1644431161781
}
```

Then, we could use below command to connect our dremio with our data source in Azure Data Lake Storage V2:
```
curl --location --request POST 'http://localhost:9047/api/v3/catalog' \
--header 'Authorization: jn5h2ds081d80a588j6doijueq' \
--header 'Content-Type: application/json' \
--data-raw '{
  "entityType": "source",
  "config": {
      "accountKind": "STORAGE_V2",
      "accountName": "****",
      "accessKey": "****",
      "enableSSL" : true
  },
  "type": "AZURE_STORAGE",
  "name": "dyvenia",
  "metadataPolicy": {
    "authTTLMs": 86400000,
    "namesRefreshMs": 3600000,
    "datasetRefreshAfterMs": 3600000,
    "datasetExpireAfterMs": 10800000,
    "datasetUpdateMode": "PREFETCH_QUERIED",
    "deleteUnavailableDatasets": true,
    "autoPromoteDatasets": false
  },
  "accelerationGracePeriodMs": 10800000,
  "accelerationRefreshPeriodMs": 3600000,
  "accelerationNeverExpire": false,
  "accelerationNeverRefresh": false
}
'
```
:::info
:bulb:Different data source might have different parameter setting, for more information, please review [the Source request introduction from dremio document](https://docs.dremio.com/software/rest-api/catalog/container-source/).
:::

### Create Physical Dataset

To create a physical dataset, we first need to know where is our dataset locate in our data source. In dremio, this information is stored as **_id**. To get this **_id**, we need to use the command like this.


```
curl --location --request GET 'localhost:9047/api/v3/catalog/by-path/dyvenia/tests' \
--header 'Authorization: jn5h2ds081d80a588j6doijueq' \
--data-raw ''
```

In my case, I want to regard a folder called "kafka" in dyvenia/tests. So I need to get the id of dyvenia/tests/kafka.

This is what I receive:
```
{
    "entityType": "folder",
    "id": "3d78e77d-349f-404a-953c-7fa7c075a080",
    "path": [
        "dyvenia",
        "tests"
    ],
    "tag": "23vXaMd5NSQ=",
    "children": [
        {
            "id": "dremio:/dyvenia/tests/csv.fmt",
            "path": [
                "dyvenia",
                "tests",
                "csv.fmt"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/csv.fmt.txt",
            "path": [
                "dyvenia",
                "tests",
                "csv.fmt.txt"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/data.csv",
            "path": [
                "dyvenia",
                "tests",
                "data.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/data2.csv",
            "path": [
                "dyvenia",
                "tests",
                "data2.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/data3.csv",
            "path": [
                "dyvenia",
                "tests",
                "data3.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/dremio",
            "path": [
                "dyvenia",
                "tests",
                "dremio"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        },
        {
            "id": "dremio:/dyvenia/tests/integration",
            "path": [
                "dyvenia",
                "tests",
                "integration"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        },
        {
            "id": "f01db6c6-48ec-4b5d-962d-5aabac169648",
            "path": [
                "dyvenia",
                "tests",
                "kafka"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        },
        {
            "id": "dremio:/dyvenia/tests/sfdc",
            "path": [
                "dyvenia",
                "tests",
                "sfdc"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        },
        {
            "id": "dremio:/dyvenia/tests/supermetrics",
            "path": [
                "dyvenia",
                "tests",
                "supermetrics"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        },
        {
            "id": "dremio:/dyvenia/tests/supermetrics_dump.csv",
            "path": [
                "dyvenia",
                "tests",
                "supermetrics_dump.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/test.csv",
            "path": [
                "dyvenia",
                "tests",
                "test.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/test1.csv",
            "path": [
                "dyvenia",
                "tests",
                "test1.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/test_data_countries.csv",
            "path": [
                "dyvenia",
                "tests",
                "test_data_countries.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/test_supermetrics_flow.csv",
            "path": [
                "dyvenia",
                "tests",
                "test_supermetrics_flow.csv"
            ],
            "type": "FILE"
        },
        {
            "id": "dremio:/dyvenia/tests/tests",
            "path": [
                "dyvenia",
                "tests",
                "tests"
            ],
            "type": "CONTAINER",
            "containerType": "FOLDER"
        }
    ]
}
```
The **_id** I am looking for is "f01db6c6-48ec-4b5d-962d-5aabac169648"

:::info
Sometimes, your _id may not be encoded and is in the path way. You could use python requests package to encode it by yourself.
```
import requests
requests.utils.quote("dremio:/dyvenia/tests/kafka", safe="")
```

You must get the encoded _id befroe the next step!
:::


Now, we are ready for the creation of physical dataset. The command is as below, 
```
curl --location --request POST 'localhost:9047/api/v3/catalog/dremio%3A%2Fkafka%2Ftests%2Fkafka' \
--header 'Authorization:jn5h2ds081d80a588j6doijueq' \
--header 'Content-Type: application/json' \
--data-raw '{
    "entityType": "dataset",
    "id": "f01db6c6-48ec-4b5d-962d-5aabac169648",
   "path": [
        "dyvenia",
        "tests",
        "kafka"
    ],
    "type": "PHYSICAL_DATASET",
    "format": {
        "type": "JSON"
    }
}'
```

### Create Virtual Dataset

The command to create historcial-virtaul dataset is as below,

```
curl --location --request POST 'localhost:9047/api/v3/catalog' \
--header 'Authorization:jn5h2ds081d80a588j6doijueq' \
--header 'Content-Type: application/json' \
--data-raw '{
  "entityType": "dataset",
  "path": [
    "sandbox",
    "history"
  ],
	"type": "VIRTUAL_DATASET",
	"sql": "SELECT  OBJECTID, PRODUCTRECIPIENTPARTYID, PRODUCTRECIPIENTPARTYUUID, PRODUCTRECIPIENTPARTYNAME, INCIDENTSERVICEISSUECATEGORYID, ID, UUID, PROCESSINGTYPECODE, PROCESSINGTYPECODETEXT, NAME, DATAORIGINTYPECODE, DATAORIGINTYPECODETEXT, ESCALATIONSTATUSCODE, ESCALATIONSTATUSCODETEXT, SERIALID, PRODUCTID, SERVICEPRIORITYCODE, SERVICEPRIORITYCODETEXT, SERVICEREQUESTUSERLIFECYCLESTATUSCODE, SERVICEREQUESTUSERLIFECYCLESTATUSCODETEXT, REQUESTEDFULFILLMENTPERIODSTARTDATETIME, REQUESTEDFULFILLMENTPERIODENDDATETIME, REQUESTEDTOTALPROCESSINGDURATION, REQUESTEDTOTALREQUESTORDURATION, REQUESTINITIALRECEIPTDATETIMECONTENT, REQUESTINPROCESSDATETIMECONTENT, REQUESTCLOSEDDATETIMECONTENT, FIRSTREACTIONDUEDATETIMECONTENT, COMPLETEDUEDATETIMECONTENT, WARRANTYSTARTDATETIMECONTENT, SALESORGANISATIONID, DIVISIONCODE, DIVISIONCODETEXT, DISTRIBUTIONCHANNELCODE, DISTRIBUTIONCHANNELCODETEXT, SALESUNITPARTYID, SERVICEEXECUTIONTEAMPARTYID, SERVICESUPPORTTEAMPARTYID, PROCESSORPARTYID, BUYERPARTYID, BUYERMAINCONTACTPARTYID, REPORTEDFORPARTYID, INSTALLATIONPOINTID, INSTALLEDBASEID, WARRANTYGOODWILLCODE, WARRANTYGOODWILLCODETEXT, ESCALATIONTIMEPOINTDATETIME, ONSITEARRIVALDUEDATETIME, ONSITEARRIVALDATETIME, SERVICESUPPORTTEAMPARTYNAME, RESOLUTIONDUEDATETIME, RESOLVEDONDATETIME, SALESTERRITORYID, RESOLUTIONDUETIMEZONECODE, RESOLUTIONDUETIMEZONECODETEXT, ESCALATIONTIMEPOINTTIMEZONECODE, ESCALATIONTIMEPOINTTIMEZONECODETEXT, ONSITEARRIVALDUETIMEZONECODE, ONSITEARRIVALDUETIMEZONECODETEXT, REQUESTINPROCESSDATETIMEZONECODE, REQUESTINPROCESSDATETIMEZONECODETEXT, WARRANTYSTARTDATETIMEZONECODE, WARRANTYSTARTDATETIMEZONECODETEXT, REQUESTCLOSEDDATETIMEZONECODE, REQUESTCLOSEDDATETIMEZONECODETEXT, REQUESTEDFULFILLMENTPERIODSTARTTIMEZONECODE, REQUESTEDFULFILLMENTPERIODSTARTTIMEZONECODETEXT, REQUESTEDFULFILLMENTPERIODENDTIMEZONECODE, REQUESTEDFULFILLMENTPERIODENDTIMEZONECODETEXT, REQUESTFINISHEDDATETIMEZONECODE, REQUESTFINISHEDDATETIMEZONECODETEXT, FIRSTREACTIONDUEDATETIMEZONECODE, FIRSTREACTIONDUEDATETIMEZONECODETEXT, RESOLVEDONTIMEZONECODE, RESOLVEDONTIMEZONECODETEXT, COMPLETEDUEDATETIMEZONECODE, COMPLETEDUEDATETIMEZONECODETEXT, REQUESTINITIALRECEIPTDATETIMEZONECODE, REQUESTINITIALRECEIPTDATETIMEZONECODETEXT, ONSITEARRIVALTIMEZONECODE, ONSITEARRIVALTIMEZONECODETEXT, SALESUNITPARTYNAME, SERVICEEXECUTIONTEAMPARTYNAME, SERVICEPERFORMERPARTYNAME, PROCESSORPARTYNAME, BUYERPARTYNAME, BUYERMAINCONTACTPARTYNAME, REPORTEDFORPARTYNAME, PRODUCTDESCRIPTION, INSTALLATIONPOINTDESCRIPTION, INSTALLEDBASEDESCRIPTION, SERVICETERMSSERVICEISSUENAME, SALESTERRITORYNAME, BUYERPARTYUUID, SERVICESUPPORTTEAMPARTYUUID, SALESUNITPARTYUUID, WARRANTYID, OBJECTSERVICEISSUECATEGORYID, CAUSESERVICEISSUECATEGORYID, ACTIVITYSERVICEISSUECATEGORYID, MAINTICKETID, CONTRACTID, REPORTEDPARTYID, SERVICEISSUECATEGORYID, SERVICEREQUESTCLASSIFICATIONCODE, SERVICEREQUESTCLASSIFICATIONCODETEXT, CREATIONDATETIME, PARTNERCONTACTPARTYID, REPORTEDPARTYNAME, CHANGEDBYCUSTOMERINDICATOR, RESPONSEBYPROCESSORDATETIMECONTENT, RESPONSEBYPROCESSORDATETIMEZONECODE, RESPONSEBYPROCESSORDATETIMEZONECODETEXT, RESPONSEBYPROCESSORDUEDATETIME, RESPONSEBYPROCESSORDUEDATETIMEZONECODE, RESPONSEBYPROCESSORDUEDATETIMEZONECODETEXT, SERVICELEVELOBJECTIVEID, SERVICELEVELOBJECTIVENAME, SERVICELEVELOBJECTIVENAMELANGUAGECODE, SERVICELEVELOBJECTIVENAMELANGUAGECODETEXT, RESPONSEBYREQUESTERDATETIMECONTENT, RESPONSEBYREQUESTERDATETIMEZONECODE, RESPONSEBYREQUESTERDATETIMEZONECODETEXT, INSTALLATIONPOINTIDV1, LASTCHANGEDATETIME, CREATIONIDENTITYUUID, LASTCHANGEIDENTITYUUID, SOLUTIONCATEGORYNAME, CAUSECATEGORYNAME, OBJECTCATEGORYNAME, INCIDENTCATEGORYNAME, CREATEDBY, LASTCHANGEDBY, SERVICEREQUESTLIFECYCLESTATUSCODE, SERVICEREQUESTLIFECYCLESTATUSCODETEXT, REQUESTASSIGNMENTSTATUSCODE, REQUESTASSIGNMENTSTATUSCODETEXT, DOCUMENTLANGUAGECODE, DOCUMENTLANGUAGECODETEXT, ENTITYLASTCHANGEDON, ETAG, SALESORDERID, MAINTENANCEPLANID, MAINTENANCEPLANNAME, ZTICKET_ML_SALESORG_KUT, ZTICKET_VMS_EMAIL_KUT, Z_TICKET_V_EGAINACTIVITYID_KUT, Z_TICKET_V_EGAINQUEUE_KUT, Z_TICKET_V_REFERENCEURL_KUT, Z_TICKET_V_REFERENCE_KUT, REPORTEDPARTYUUID, BUYERMAINCONTACTPARTYUUID, PROCESSORPARTYUUID, SERVICEEXECUTIONTEAMPARTYUUID,ROW_NUMBER()OVER(partition by OBJECTID order by LASTCHANGEDATETIME DESC) as rn FROM dyvenia.tests.kafka",
	"sqlContext": ["dyvenia","tests"]
}'
```
The command to create latest-virtaul dataset is as below,

```
curl --location --request POST 'localhost:9047/api/v3/catalog' \
--header 'Authorization: jn5h2ds081d80a588j6doijueq' \
--header 'Content-Type: application/json' \
--data-raw '{
  "entityType": "dataset",
  "path": [
    "sandbox",
    "latest"
  ],
	"type": "VIRTUAL_DATASET",
	"sql": "select * from sandbox.his where rn=1",
	"sqlContext": ["sandbox"]
}'
```

## Step 9: Use PowerBI for business analysis

In Dremio, you could use the PowerBI and Tableau to visualize your dataset by clicking the button above with ODBC connector. (You will need to install PowerBI and Tableau first)


![](https://i.imgur.com/M0VgPPN.png)
![](https://i.imgur.com/JW2Att8.png)

