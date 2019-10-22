# Sierra Wireless Innovation Summit 2019 - Cloud Workshop

## Purpose
Build a solution that performs anomaly detection and data visualization of sensor's data in near realtime.
This solution also provide the ability to configure live alerting depending on the anomaly score.
The sensor data is also backed up in an Amazon S3 bucket for further use.

## Prerequisites

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

### ...On your own
Please follow this guide 
