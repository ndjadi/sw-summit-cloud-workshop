# Sierra Wireless Innovation Summit 2019 - Cloud Workshop

## Purpose
Build a solution that performs anomaly detection and data visualization of sensor's data in near realtime.
This solution also provide the ability to configure live alerting depending on the anomaly score.
The sensor data is also backed up in an Amazon S3 bucket for further use.

The anomaly detection relies on the Random Cut Forest algorithm

## Prerequisites
Here is an example of the payload data that it is expected from the application :
```json
{
    "Data": {
      "deviceId": "mangoh_device",
      "creationDate": 1571735678858,
      "generatedDate": 1571735677678,
      "accel": {
          "x": -0.688896,
          "y": -0.742716,
          "z": 9.868196
      },
      "light": 146,
      "temp": 32.457031,
      "location": "48.875004,2.235794"
    },
    "PartitionKey": 1  
}
```
Please make sure to configure the appropriate edge and cloud action in Octave to send data in the format described above to AWS. During the workshop, a high frequency (every 2 seconds) is recommended. (This will speed up the Random Cut Forest Algorithm learning phase.


## Architecture

![Image description](stack/sw-architecture-diagram.png)


## Step by step guide
### ...During Sierra Wireless Innovation Summit
Some components of the architecture have been deployed ahead of the workshop (in order to save some time).
We will primarily focus on building The Kinesis Data Analytics and the Kinesis Data firehose components.
#### 1. Connect to your temporary AWS accounts
Go to https://dashboard.eventengine.run and enter the hash number you've been provided with and open the AWS Console
Welcome to your temporary AWS account.
#### 2. Retrieve your Amazon API gateway endpoint
In the AWS console, navigate to CloudFormation. On the deployed stack output, copy the API Gateway endpoint, and make sure you are using this endpoint in your Octave cloud actions to send data to AWS.
#### 3. Prepare Elasticsearch and Kibana
Navigate to Elasticsearch service on the AWS Console and browse the Kibana endpoint.
Go to Dev tools and create a mapping template :

```json
PUT _template/template_mangoh
{
  "index_patterns": ["mango*"],
  "mappings": {
    "_source": {
      "enabled": true
    },
    "properties": {
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```
Now navigate to Index management, click on saved objects and import the file elasticsearch/sw-kibana-objects.json. This will import the index pattern, visualizations and the dashboard.

#### 4. Configure Kinesis Data Analytics :
Navigate to Kinesis Data Analytics and create a new SQL application.
Connect it to the "sw-stream" Kinesis Data Stream and use this query :

```sql
CREATE OR REPLACE STREAM "TEMP_STREAM" (
    "device_id"      VARCHAR(32),
    "creation_date"  TIMESTAMP,
    "generated_date"  TIMESTAMP,
    "light_sensor"   INTEGER,
    "acc_x" DOUBLE,
    "acc_y" DOUBLE,
    "acc_z" DOUBLE,
    "temp_sensor" DOUBLE,
    "location" VARCHAR(128),
    "ANOMALY_SCORE"  DOUBLE,
    "ANOMALY_EXPLANATION" VARCHAR(20480));
 -- Creates an output stream and defines a schema
 CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" (
    "device_id"      VARCHAR(32),
    "creation_date"  VARCHAR(20),
    "generated_date"  VARCHAR(20),
    "light_sensor"   INTEGER,
    "acc_x" DOUBLE,
    "acc_y" DOUBLE,
    "acc_z" DOUBLE,
    "temp_sensor" DOUBLE,
    "location" VARCHAR(128),
    "ANOMALY_SCORE"  DOUBLE,
    "ANOMALY_EXPLANATION" VARCHAR(20480));
 
 -- Compute an anomaly score for each record in the source stream
 -- using Random Cut Forest
 CREATE OR REPLACE PUMP "STREAM_PUMP" AS INSERT INTO "TEMP_STREAM"
    SELECT STREAM
    "deviceId" as "device_id",
    "creationDate" as "creation_date",
    "generatedDate" as "generated_date",
    "light" as "light_sensor",
    "x" as "acc_x",
    "y" as "acc_y",
    "z" as "acc_z",
    "temp" as "temp_sensor",
    "location",
    "ANOMALY_SCORE",
    "ANOMALY_EXPLANATION" FROM TABLE(RANDOM_CUT_FOREST_WITH_EXPLANATION(CURSOR(
        SELECT STREAM 
        "deviceId",
        TO_TIMESTAMP("creationDate") as "creationDate",
        TO_TIMESTAMP("generatedDate") as "generatedDate",
        "light",
        "x",
        "y",
        "z",
        "temp",
        "location"
        FROM "SOURCE_SQL_STREAM_001"), 10, 256, 10000, 10, true));

-- Sort records by descending anomaly score, insert into output stream
 CREATE OR REPLACE PUMP "OUTPUT_PUMP" AS INSERT INTO "DESTINATION_SQL_STREAM"
    SELECT STREAM
    "device_id",
    TIMESTAMP_TO_CHAR('yyyy-MM-dd', "creation_date")||'T'||TIMESTAMP_TO_CHAR('HH:mm:ss', "creation_date"),
    TIMESTAMP_TO_CHAR('yyyy-MM-dd', "generated_date")||'T'||TIMESTAMP_TO_CHAR('HH:mm:ss', "generated_date"),
    "light_sensor",
    "acc_x",
    "acc_y",
    "acc_z",
    "temp_sensor",
    "location",
    "ANOMALY_SCORE", 
    "ANOMALY_EXPLANATION"
    FROM "TEMP_STREAM"
    ORDER BY FLOOR("TEMP_STREAM".ROWTIME TO SECOND), ANOMALY_SCORE DESC;

```

#### 5. Configure a Kinesis Data Firehose delivery stream :
The Kinesis Data Firehose delivery stream will send data to ElasticSearch in near real-time. It will also dump all the data in an S3 bucket.

#### 6. Send Kinesis Data Analytics output to targets :
DESTINATION_SQL_STREAM should be sent to a Kinesis Data Firehose delivery stream created in step 5.
ALERTING_SQL_STREAM should be sent to the "sw-lambda-notification" Lambda function (to fire alerts)

#### 7. Vizualisation :
Go to Kibana and display the dashboard.

### ...On your own
Please follow this guide
