#### lylambdadeepd
#####Event-driven Fundamentals
push:event resources invoke lambda function  
poll: aws lambda polls the event source, invoke lambda func if theres a new event (used by dynamodb and kinesis  
event source mapping: mapping event source ->lambda
######Understanding Lambda Limits
limit: 100 concurrent functions
#####2
######Getting Started With Lambda
s3->lylambdadeep->(images,processed)  
choose lambda->blueprint->s3->s3 get  
config: event type : Object Created (All) prefix: images/  
Name: objectMeatdata  

Event source: in tab "event source",set state to enabled, this is equivalent to set property in S3.(s3->bucket->event->lambda)

function
```
'use strict';
console.log('Loading function');

let aws = require('aws-sdk');
let s3 = new aws.S3({ apiVersion: '2006-03-01' });

exports.handler = (event, context, callback) => {
    //console.log('Received event:', JSON.stringify(event, null, 2));

    // Get the object from the event and show its content type
    const bucket = event.Records[0].s3.bucket.name;
    const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '));
    const params = {
        Bucket: bucket,
        Key: key
    };
    s3.getObject(params, (err, data) => {
        if (err) {
            console.log(err);
            const message = `Error getting object ${key} from bucket ${bucket}. Make sure they exist and your bucket is in the same region as this function.`;
            console.log(message);
            callback(message);
        } else {
            console.log('CONTENT TYPE:', data.ContentType);
            console.log('CONTENT LAST:', data.LastModified);
            console.log('CONTENT LENGTH:', data.ContentLength);
            callback(null, data.ContentType);
        }
    });
};
```
######Python Function Walkthrough
Name: pythonObjectMeatdata  prefix: processed/ ,donot to add a event to s3
```
from __future__ import print_function

import json
import urllib
import boto3

print('Loading function')

s3 = boto3.client('s3')


def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))

    # Get the object from the event and show its content type
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.unquote_plus(event['Records'][0]['s3']['object']['key']).decode('utf8')
    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        print("CONTENT TYPE: " , context.memory_limit_in_mb)
        print("Request id: " , context.aws_request_id)
        print("CONTENT TYPE: " , response['ContentType'])
        print("CONTENT length: " ,response['ContentLength'])
        print("last modified: " , response['LastModified'])
        return response['ContentType']
    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e
```
######Creating Lambda Functions With The CLI
```
aws iam list-roles  //get appropriate arn
```
then
```
aws lambda create-function --region us-east-1 --function-name kk --zip-file
fileb://index.zip --handler index.handler --runtime nodejs --description "test"
--role arn:aws:iam::64412686:role/lambda_s3_exec_role
```
Or from S3 (--code insteadof --zip-file
```
aws lambda create-function --region us-east-1 --function-name kk --code S3Bu
cket=name,S3Key=index.zip --handler index.handler --runtime nodejs -
-description "test" --role arn:aws:iam::64412686:role/lambda_s3_exec_role --timeout 2 --memory-size 128
```
######Managing Push Events And Permissions With The CLI
add permission to a function:
```
aws lambda add-permission --function-name kk --statement 1 --principal s3.am
azonaws.com --action lambda:Invokefunction --source-arn arn:aws:s3:::lylambdadee
pdive --source-account 64412686 --region us-east-1
{
```
get policy
```
aws lambda get-policy --function-name kk
```
Add another
```
aws lambda add-permission --function-name kk --statement 3 --principal 64412686 --action lambda:Invokefunction --region us-east-1
```
get notification (we need to add lambda notification later)
```
aws s3api get-bucket-notification-configuration --bucket lylambdadeepdive
```
add notification, first create a file lambdaconfig.json
```
{
"LambdaFunctionConfigurations":[{
  "LambdaFunctionArn":"arn:aws:lambda:us-east-1:64412686:function:kk",
  "Events":[
    "s3:ObjectCreated:*"
  ],
  "Filter":{
    "Key":{
    "FilterRules":[{
    "Name":"prefix",
    "Value":"test/"}]
  }}}
]
}
```
then
```
aws lambda list-functions --query Functions[4].{Arn:FunctionArn} --region us-east-1
```
set config
```
aws s3api put-bucket-notification-configuration --bucket lylambdadeepdive --notification-configuration file:///lambdaconfig.json
```
######Testing Functions With The CLI
create test
```
{
"Records":[
{
"eventVersion":"2.0",
"eventSource":"aws:s3",
"awsRegion":"us-east-1",
"eventName":"1999-01-01T00:00:00.0001",
"eventName":"ObjectCreated:Put",
"userIdentity":{
"PrincipalId":"123"
},
"requestParameters":{
"sourceIPAddress":"127.0.0.1"},
"responseElements":{
"x-amz-request-id":"123",
"x-amz-id-2":"22z"},
"s3":{
"s3SchemaVersion":"1.0",
"configurationId":"testConfigRule",
"bucket":{
"name":"lylambdadeepdive",
"ownerIdentity":{
"principleId":"ke"},
"arn":"arn:aws:s3:::lylambdadeepdive"
},
"object":{
"key":"images/uploaded.png",
"size":1000,
"eTag":"123",
"versionId":"1213"}
}
}
]
}
```
dryrun: should return code 204.note: change node to v4
```
aws lambda invoke --function-name kk --payload file:///test.json --invocation-type DryRun output.json
```
real run:
```
aws lambda invoke --function-name kk --payload file:///test.json output.json
```
check output.json,should show
```
image/png
```
######Managing Pull Events And Event Source Mappings With The CLI
file
```
{
"Version":"2012-10-17"
"Statement":[
{
"Effect":"Allow",
"Principal":{"Service":"lambda.amazonaws.com"},
"Action":"sts:AssumeRole"
}
]}
```
createrole
```
aws iam create-role --role-name kkrole --assume-role-policy-document file:///lambdarole.json
```
(to be conti)
######Retrieving Lambda CloudWatch Logs From The CLI
```
aws logs describe-log-groups --region us-east-1
aws logs describe-log-streams --log-group-name /aws/lambda/kk --region us-east-1
aws logs describe-log-streams --log-group-name /aws/lambda/kk --region us-east-1 --log-stream-name-prefix 2016/05/29
aws logs describe-log-streams --log-group-name /aws/lambda/kk --region us-east-1 --output text --query logStreams[*].logStreamName
```
more complicated
```
log_streams =$(aws logs describe-log-streams --log-group-name /aws/lambda/kk --region us-east-1 --output text --query logStreams[*].logStreamName) && for log_stream in $log_streams; do aws logs get-log-events --log-group-name /aws/lambda/kk --log-stram-name $log-stream- --output text --query events[*].message; done|less
```
######Test with dynamodb
create a dynamodb, key =id  
create a lambda function using python,
```
from __future__ import print_function

import json
import boto3

print('Loading function')


def lambda_handler(event, context):
    #print("Received event: " , json.dumps(event, indent=2))
    operation = event['operation']
    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])
    operations ={
        'create':lambda x:dynamo.put_item(**x)
        }
    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('error')
```

using api gateway:: body
```
{
    "operation":"create",
    "tableName":"lylambdadeepdive",
    "payload":{
    "Item":{
        "id":"1",
        "name":"ke"
    }
}
}
```
######Test nodejs local (gulp)

