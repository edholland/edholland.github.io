---
layout: post
title: Serverless HyperLogLog backed with AWS Lambda, API Gateway, DynamoDB and Kinesis
---

## Introduction
I recently read an [article](http://blog.gingerlime.com/2016/a-scaleable-ab-testing-backend-in-100-lines-of-code-and-for-free/)
describing an A/B testing platform implemented on AWS
Lambda backed with a Redis HyperLogLog backend, however I was left with the
feeling that we could take it one step further:  A serverless HyperLogLog
implementation backed with DynamoDB and a Kinesis write buffer.

# What is HyperLogLog
HyperLogLog is an algorithm for estimating the cardinality of a multiset (a set
which allows repetition) in a fixed memory space. An optimal system can
estimate with a 2% accuracy the cardinality of billions of items with less than
2kB of memory. Further details can be found in the [orginal paper](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
, and in Google's [version](http://research.google.com/pubs/pub40671.html)
corrected for low cardinality, and in a [blog post](http://antirez.com/news/75) detailing the Redis implementation.

## Design

# Use Case
As we are targeting web analytics use cases we should provide a REST API to
allow simple integration into a diverse variety of existing web technologies.
We must also consider the performance characteristics we want to optimise for.
Many web analytics tools are embedded directly into the end user's browser and
sent asynchronous events to the underlying platform. This results in a very
write-heavy system, as leads tend to be limited to result dashboards typically
only viewed by a small cohort of users.

# AWS Lambda + API Gateway
To expose a REST API from an AWS Lambda function we can utilise API gateway to
translate HTTP calls into events which Lambda can respond to. API gateway uses
the OpenAPI specification language (aka Swagger) to define the mapping between
the HTTP endpoint and the Lambda function, thankfully AWS provides the following
template.

```
##  See http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html
##  This template will pass through all parameters including path, querystring, header, stage variables, and context through to the integration endpoint via the body/payload
#set($allParams = $input.params())
{
    "body-json" : $input.json('$'),
    "params" : {
    #foreach($type in $allParams.keySet())
        #set($params = $allParams.get($type))
        "$type" : {
            #foreach($paramName in $params.keySet())
            "$paramName" : "$util.escapeJavaScript($params.get($paramName))"
            #if($foreach.hasNext),
            #end
        #end
    }
    #if($foreach.hasNext),#end
        #end
    },
    "stage-variables" : {
        #foreach($key in $stageVariables.keySet())
            "$key" : "$util.escapeJavaScript($stageVariables.get($key))"
                #if($foreach.hasNext),#end
            #end
    },
    "context" : {
        "account-id" : "$context.identity.accountId",
        "api-id" : "$context.apiId",
        "api-key" : "$context.identity.apiKey",
        "authorizer-principal-id" : "$context.authorizer.principalId",
        "caller" : "$context.identity.caller",
        "cognito-authentication-provider" : "$context.identity.cognitoAuthenticationProvider",
        "cognito-authentication-type" : "$context.identity.cognitoAuthenticationType",
        "cognito-identity-id" : "$context.identity.cognitoIdentityId",
        "cognito-identity-pool-id" : "$context.identity.cognitoIdentityPoolId",
        "http-method" : "$context.httpMethod",
        "stage" : "$context.stage",
        "source-ip" : "$context.identity.sourceIp",
        "user" : "$context.identity.user",
        "user-agent" : "$context.identity.userAgent",
        "user-arn" : "$context.identity.userArn",
        "request-id" : "$context.requestId",
        "resource-id" : "$context.resourceId",
        "resource-path" : "$context.resourcePath"
    }
}
```

# Python HyperLogLog
To avoiding having to implement our own version of HyperLogLog we can lean on
the existing module found at https://github.com/svpcom/hyperloglog. This has
already implemented the main HyperLogLog and Google's correction for sets of low
cardinality, and provides an API to merge several HyperLogLog objects.

# Persistence
DynamoDB is Amazon's NoSQL database service, we will use the database to persist
the state of our HyperLogLog. We will store a binary representation of the Python
HyperLogLog object, although this is less efficient that an implementation which
just stored the HyperLogLog binary the additional cost of a few kb of storage is
minimal when compared to the development costs of a native implementation.
As we are building for a very write-heavy system if we were to attempt to update
the HyperLogLog for each write we would run into concurrency/consistency issues
so we provide a write buffer behind the HTTP endpoint.
An HTTP write request will not attempt to directly update the HyperLogLog,
instead the the data item will be pushed into a Kinesis stream. In the Kinesis
stream we collect records into 60 second, or 10,000 record batches, before making
a single write operation for everything within this batch
Adding this Kinesis stream allows us to horizontally scale our write capacity
by simply adding more Kinesis shards, each allowing around 1 mb/s write capacity.
To avoid coordination between shards we store a separate HyperLogLog object for
each shard in the stream, as we then know only 1 backend processing Lambda will
ever attempt to update to each HyperLogLog object. The read layer simply merges
each shard's HyperLogLog object before returning a result

## Implementation
The implementation consists of two parts, a front end API endpoint that provides
functions to write new items via the Kinesis stream and to read the cardinality
from DDB and secondly a backend processing module which consumes Kinesis streams.
The DDB table uses the HyperLogLog name as the DDB hash key, and the shard name as
the range key. This allows us to get the query the relevant HyperLogLog binary
objects without resorting to a table scan or secondary indexes

```python
from __future__ import print_function

import boto3
import json
from time import time
import hyperloglog
import pickle
from boto3.dynamodb.conditions import Key, Attr

print('Loading function')

def lambda_handler(event, context):
    print("Received event: " + json.dumps(event, indent=2))

    operation = event['context']['http-method']
    name = event.get('params').get('path').get('name')

    if name:
        kinesis = boto3.client('kinesis')
    else:
        raise ValueError('Must provide log name')
    operations = {
        'POST': lambda x: kinesis.put_record(StreamName=name,
                                             PartitionKey=str(time()),
                                             Data= x.get('body-json').get('data')),
        'GET':  lambda x: get_card(name, x)
    }

    if operation in operations:
        return operations[operation](event)
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))

def get_card(name, event):
    dynamo = boto3.resource('dynamodb').Table(name)
    items = dynamo.query(KeyConditionExpression=Key('name').eq(name),
                         ProjectionExpression='bin')['Items']
    hll = hyperloglog.HyperLogLog(0.01)
    [ hll.update(pickle.loads(str(item['bin']))) for item in items ]
    return hll.card()
```

This front-end Lambda provides two functions. A GET operation to retrieve an
estimate from an existing HyperLogLog using the URL `http://<url>/hyperloglog/{name}`
and a write operation to add new records to the HyperLogLog called by POSTing a
JSON object such as `{ "data": "http://some.url/data/point"}` to the URL
`http://<url>/hyperloglog/{name}`.

```python
from __future__ import print_function

import boto3
import base64
import json
import hyperloglog
import pickle
from  boto3.dynamodb.types import Binary, TypeDeserializer

print('Loading function')

def lambda_handler(event, context):
    print("Received event: " + json.dumps(event, indent=2))
    record     = event['Records'][0]
    streamName = record['eventSourceARN'].split(":")[-1].split("/")[-1]
    shardId    = record['eventID'].split(":")[0]
    dynamo     = boto3.resource('dynamodb').Table(streamName)
    try:
        item_bin = dynamo.get_item(Key={'name':streamName,
                                        'shard':shardId},
                                   ProjectionExpression='bin')['Item']['bin']
        hll = pickle.loads(str(item_bin))
        print("loaded from ddb")
    except KeyError:
        hll = hyperloglog.HyperLogLog(0.01)
    for record in event['Records']:
        # Kinesis data is base64 encoded so decode here
        payload = base64.b64decode(record['kinesis']['data'])
        print("adding '{}'".format(payload))
        hll.add(payload)

    res = dynamo.put_item(Item={'name': streamName,
                                'shard': shardId,
                                'bin': Binary(pickle.dumps(hll, 2))})

    print('Successfully processed {} records. Card: {}'.format(len(event['Records']), hll.card()))
    return "done"
```

The backend processing module consumes a write buffer (as a Kinesis stream)
adding all records in a batch in a single read/write round trip to DDB.
