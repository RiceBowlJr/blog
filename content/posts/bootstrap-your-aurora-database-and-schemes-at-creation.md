---
#Categories:
#- lambda
#- amazon
Description: "Bootstrap your Aurora schemes and users at creation."
Tags:
- lambda
- amazon
title: "Bootstrap Your Aurora Schemes ans users at Creation"
date: 2018-06-01T18:26:54+02:00
draft: false
---

# Context

Since I like to automate kind of everything I can, I struggled with AWS Aurora
for a client who needed multiple schemes and a non-root user. With
CloudFormation (or even through the web console), you can only launch your
Aurora database with one scheme and the root user.<br>
Some may say that we don't launch an Aurora database that often, but hey, in
some cases, you do. For example for a fast growing start up (web app editor)
which onboard new customers every single month, and whose customers require to
be on a dedicated infrastructure (yes, some still does).

To do so I struggled with **CloudFormation notifications**, **SNS** and
**Lambda**.

# A brief history of search

Basicaly, I just launch an Aurora cluster with CloudFormation. So I first want
to know when my cluster is ready so I can launch a Lambda function which will
bootstrap my schemes and low privileges users.<br>
After launching a first Aurora cluster and watching closely when CloudFormation
told me `CREATION_COMPLETE` so I could connect to my [Mysql] database, I
found out that CloudFormation wait until the resource is completly ready, so
when the Aurora cluster generates an endpoint and switch to `Ready` status.<br>
Perfect! Exactly what I was looking for, I can do something with
CloudFormation, so it can told me that the database is ready.

Next step, I go to CloudWatch Events to look for a CloudFormation Event to
trigger my Lambda... *Doh!* No CloudFormation Event, or at least, that's not
pretty clear.<br>
After some research, I found out that we can generate notifications and put those as
messages in a SNS queue. You simply have to add the `--notification-arns
<my-arns>` when launching the CloudFormation stack.

So lets destroy everything and recreate all the things. Wait, my Lambda, it
needs to run inside my VPC so I can attribute it a Security Group and allow it
to access to my Aurora cluster, easy, that's a feature made by AWS, you just
need to be careful on how often you trigger your Lambda, because you don't want to use
all your IPs available in your subnets, and forbid any of your application
running on EC2 to launch.

# Architecture

So lets summarize everything, we need to:

  1. Create a SNS queue in a first stack
  2. Create a Lambda function which will be triggered by the previous SNS
queue. Lets create them in the same stack
  3. Create an Aurora cluster within a separate CloudFormation stack, so we can
enable `--notification-arns` option on this stack

I tried a quick shema so it can be a little more visual:
<center>![Architecture](https://static.blog.ricebowljr.cc/cloudformation-sns-lambda-resource-bootstrap.png)</center>

As you can see, the workflow is the following (lets assume we already have created the SNS queue and Lambda function):

  1. Create the Aurora cluster with the `--notification-arns` option, and as value,
     the SNS ARN
  2. At each event, CloudFormation sends a notification to the SNS queue
  3. This notification trigger a Lambda function
  4. The Lambda function queries the CloudFormation API to see if the stack
     is in `CREATE_COMPLETE` state. If not, the process loops there, with
     the next CloudFormation notification
  5. The Lambda function catches a `CREATE_COMPLETE` status, so it can start the
     script that will create the schemes and the user on the database

# Code

Well, lets prove that it is really doable.

First, we need to launch the SNS Queue and the Lambda function. We have to create
the Security Groups and export them so we can reuse them in other stacks:

```
Resources:

  InitDbSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: initdb-sg
      GroupDescription: Created by CloudFormation
      VpcId: !ImportValue VPCid

  RDSClusterSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: aurora-sg
      GroupDescription: Created by CloudFormation
      VpcId: !ImportValue VPCid
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref InitDbSG

Outputs:
  RDSClusterSG:
    Description: RDS Aurora Cluster Security Group ID
    Value: !GetAtt RDSClusterSG.GroupId
    Export:
      Name: RDSClusterSG

  InitDbSG:
    Description: Lambda InitDB Security Group ID
    Value: !GetAtt InitDbSG.GroupId
    Export:
      Name: InitDbSG
```

And now the SNS queue and Lambda function:

```
Resources:

  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        S3Bucket: templates.osones.com
        S3Key: lambda/initdb.zip
      Description: Database Init
      Environment:
        Variables:
          STACK_NAME: osones-blog-demo-aurora
      FunctionName: InitDb
      Handler: "index.handler"
      Role: !GetAtt LambdaInitDbServiceRole.Arn
      Runtime: nodejs6.10
      Timeout: 30
      VpcConfig:
        SecurityGroupIds:
          - !ImportValue InitDbSG
        SubnetIds:
          - !ImportValue PrivateSubnet1
          - !ImportValue PrivateSubnet2
          - !ImportValue PrivateSubnet3

  LambdaInitDbServiceRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument: |
        {
            "Statement": [{
                "Effect": "Allow",
                "Principal": { "Service": [ "lambda.amazonaws.com" ]},
                "Action": [ "sts:AssumeRole" ]
            }]
        }
      Policies:
        - PolicyName: root
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Resource:
                  - arn:aws:logs:*:*:*
                Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
              - Resource: "*"
                Effect: Allow
                Action:
                  - cloudformation:DescribeStacks
              - Resource: "*"
                Effect: Allow
                Action:
                  - ec2:CreateNetworkInterface
                  - ec2:DescribeNetworkInterfaces
                  - ec2:DeleteNetworkInterface

  PermissionForSnsToInvokeLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LambdaFunction
      Action: lambda:InvokeFunction
      Principal: sns.amazonaws.com
      SourceArn: !Ref SnsTopic

  SnsTopic:
    Type: AWS::SNS::Topic
    Properties:
      DisplayName: cloudformation-notify
      Subscription:
        -
          Endpoint: !GetAtt LambdaFunction.Arn
          Protocol: lambda

Outputs:
  SnsTopicArn:
    Description: ARN of SNS Topic for this Lambda
    Value: !Ref SnsTopic
    Export:
      Name: snstopic-arn
```

As you can see, the SNS Topic needs a special permission to invoke the Lambda
function, and that's something you don't have to do within the web console.<br>
You may want to retrieve the root user and password from the parameter store so
they don't appear in clear text in your Lambda.<br>
You can also see that we grab the Lambda code from an Osones public bucket to
make your life easier, but for you, here is the code:
```
'use strict';
const AWS = require("aws-sdk");
const mysql = require("mysql");
console.log('Loading function');

exports.handler = (event, context, callback) => {

    console.log(JSON.stringify(event));

    if (event.Records[0]['Sns'] && event.Records[0]['Sns'].Subject === "AWS CloudFormation Notification") { //// we need to   make sure the event is coming from cloudformation
       const information = {};
       const quoteRegexp = new RegExp(/'(.*)'/);
          if (event.Records[0]['Sns'] && event.Records[0]['Sns'].Subject === "AWS CloudFormation Notification") { //// we     need to make sure the event is coming from cloudformation
            event.Records[0]['Sns'].Message.split("\n").forEach(function(line) {
              if (line.indexOf('=') !== -1) {
               const [key, value] = line.split("=");
               information[key] = value.match(quoteRegexp)[1];
             }
          });
        }
        console.log(information);
        if (information.ResourceStatus === "CREATE_COMPLETE" && information.LogicalResourceId === process.env.STACK_NAME) {               console.log("Event found");
            console.log("Now lets found Aurora Endpoint");
            var params = {
              StackName: information.LogicalResourceId
            };
            var cloudformation = new AWS.CloudFormation();
            cloudformation.describeStacks(params, function(err, data) {
              if (err) console.log(err, err.stack); // an error occurred
              else {
                console.log("Next line is the get data");
                console.log(data);           // successful response
                console.log("Previous line was the last line of the get data");
                if(data.Stacks.length > 0) {
                  var auroraEndpoint = null;
                  console.log(data.Stacks[0]);
                  var stack = data.Stacks[0];
                  var stackName = stack.StackName;
                  var stackStatus = stack.StackStatus;
                  if (stackName === information.LogicalResourceId && stackStatus === "CREATE_COMPLETE") {
                    for (let index in stack.Outputs) {
                      if (stack.Outputs[index].OutputKey === "AuroraEndpointDns") {
                        auroraEndpoint = stack.Outputs[index].OutputValue;
                        break;
                      }
                    }
                  }
                  //Here exec the requests to init the DB
                  console.log("Aurora endpoint is: "+auroraEndpoint);
                  var connection = mysql.createConnection({
                    host     : auroraEndpoint,
                    user     : 'root',
                    password : 'osones2018'
                  });

                  connection.connect();

                  connection.query('CREATE DATABASE `my-new-scheme` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;',       function(err, rows, fields) {
                    if (err) throw err;
                    console.log(rows);
                  });
                  connection.query('CREATE USER ricebowljr IDENTIFIED BY \'myNotSoSecuredPassword\';', function(err, rows, fields) {
                    if (err) throw err;
                    console.log(rows);
                  });
                  connection.query('GRANT ALL PRIVILEGES ON `my-new-scheme`.* TO \'ricebowljr\'@\'%\';', function(err, rows, fields) {
                    if (err) throw err;
                    console.log(rows);
                  });

                  connection.end();
                }
              }
            });
        }
    }
    callback(null, event);
};

```

Finally, we generate the Aurora cluster with the following template:

```
Resources:
  RDSCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      MasterUsername: !Ref Username
      MasterUserPassword: !Ref Password
      Engine: aurora-mysql
      DBClusterParameterGroupName: !Ref RDSDBClusterParameterGroup
      DBSubnetGroupName: !Ref DBSubnetGroupName
      VpcSecurityGroupIds:
        - !ImportValue RDSClusterSG

  RDSDBInstance1:
    Type: AWS::RDS::DBInstance
    Properties:
      DBParameterGroupName: !Ref RDSDBParameterGroup
      Engine: aurora-mysql
      DBClusterIdentifier: !Ref RDSCluster
      PubliclyAccessible: 'false'
      DBInstanceClass: !Ref InstanceType

  RDSDBClusterParameterGroup:
    Type: AWS::RDS::DBClusterParameterGroup
    Properties:
      Description: CloudFormation made Aurora Cluster Parameter Group
      Family: aurora-mysql5.7
      Parameters:
        time_zone: UTC

  RDSDBParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Description: CloudFormation made Aurora Parameter Group
      Family: aurora-mysql5.7

  DBSubnetGroupName:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: DBSubnetGroup
      SubnetIds:
        - { "Fn::ImportValue" : {"Fn::Sub": "PrivateSubnet1" } }
        - { "Fn::ImportValue" : {"Fn::Sub": "PrivateSubnet2" } }
        - { "Fn::ImportValue" : {"Fn::Sub": "PrivateSubnet3" } }
```


As you can see, you need some parameters and other stuff to launch this
resource. You may want to get your **root user** and **password** in the
Parameter Store, or at least set the CloudFormation parameter `NoEcho: true` so
they are not shown in the stack parameters within the web console or AWS CLI.

Carefull when launching this stack, you need to name it accordingly to the name
set within the Lambda function, which in our case is `osones-blog-demo-aurora`.
Otherwise the function won't find the stack and won't be able to check the
status and retrive the Aurora endpoint.

And with all that your good to go. You're free to delete the SNS/Lambda stack
since it is only one time usage.

Happy automation!
