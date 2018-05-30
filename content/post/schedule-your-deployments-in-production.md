---
#Categories:
#- lambda
#- amazon
Description: "Automate your deployments in production with AWS Lambda and
CodePipeline"
Tags:
- lambda
- amazon
date: 2018-05-30T19:22:19+02:00
draft:  true
title: "Schedule Your Deployments in Production"
---

This article was originaly published on [Osones
blog](https://blog.osones.com/en).


## Some context

Some of our clients asked us for a feature in their CodePipeline CI/CD that is not native to AWS: they need to deploy in production but on scheduled, for example only on mondays at 9am.<br />

As I said, that feature is not native to AWS CodePipeline service, so we made a little trick, using **AWS Lambda** and **CloudWatch Events**. To simplify the deployment, as usual we then created an **AWS CloudFormation template** to automate things a bit.<br />

Let's dive in!

## The solution architecture

The architecture is pretty basic. We add a step in our Pipeline, directly into the last stage, just before the last step which is the production deployment. The new step is an **Manual Approval**, we volontarily skip this creation process thus it is basic to do, just keep in mind that you don't want AWS to spam your mailbox so don't add an SNS Notification to the Approval.<br />

This Approval is now the wall that prevent to deploy to production.<br />

The other part is the CloudWatch Event that trigger the Lambda function. It is configured with a scheduled cron rule that trigger the Lambda on mondays at 9am.<br />
Then the Lambda function: a JavaScript script that interact with the Pipeline. It gather infos about the pipeline, because to approve the approval, a token is needed, and then approve the approval.

<center><img src="images/AutomaticCodePipelineApprovalArchitecture.png" alt="Architecture" width="800" align="middle"></center>

## Deploy the solution

As said before, at Osones, we love automate things. So we made a CloudFormation template out of the solution, and it is available on [our GitHub page](https://github.com/Osones/cloud-infra/blob/master/aws/lambda/codepipeline-scheduled-approval/00-infra-codepipeline-scheduled-approval.yml). So lets take a tour of the template.<br />

```
  LambdaFunction:
    Type: "AWS::Lambda::Function"
    Properties:
      Code:
        ZipFile:
          !Join
            - ' '
            - - 'var AWS = require("aws-sdk");'
              - 'exports.handler = (event, context, callback) => {'
              - '  var paramsGet = {'
              - '    name: process.env.PIPELINE_NAME'
              - '  };'
              - '  var codepipeline = new AWS.CodePipeline();'
              - '  codepipeline.getPipelineState(paramsGet, function(err, data) {'
              - '    if (err) console.log(err, err.stack);'
              - '    else {'
              - '      if (data.stageStates.length > 0) {'
              - '        var token = null;'
              - '        for (let stage of data.stageStates) {'
              - '          var stageName = stage.stageName;'
              - '          if (stageName === process.env.STAGE_NAME) {'
              - '            var actionStates = stage.actionStates;'
              - '            for (let actionState of actionStates) {'
              - '              var actionName = actionState.actionName;'
              - '              if (actionName === process.env.APPROVAL_NAME) {'
              - '                token = actionState.latestExecution.token;'
              - '                break;'
              - '              }'
              - '            }'
              - '          }'
              - '        }'
              - '        var paramsPut = {'
              - '          actionName: process.env.APPROVAL_NAME,'
              - '          pipelineName: process.env.PIPELINE_NAME,'
              - '          result: {'
              - '            status: "Approved",'
              - '            summary: "Scheduled approval by Lambda."'
              - '          },'
              - '          stageName: process.env.STAGE_NAME,'
              - '          token: token'
              - '        };'
              - '        if (token != null) {'
              - '          codepipeline.putApprovalResult(paramsPut, function(err, data) {'
              - '            if (err) console.log(err, err.stack);'
              - '            else console.log(data);'
              - '          });'
              - '        }'
              - '      }'
              - '    }'
              - '    callback(null, "Approved");'
              - '  })'
              - '};'
      Description: Automatic CodePipeline Approval
      Environment:
        Variables:
          PIPELINE_NAME: !Ref PipelineName
          STAGE_NAME: !Ref StageName
          APPROVAL_NAME: !Ref ApprovalName
      FunctionName: AutomaticCodePipelineApproval
      Handler: "index.handler"
      Role: !GetAtt LambdaAutoApprovalServiceRole.Arn
      Runtime: nodejs6.10
```

We first declare the code within a `Fn::Join` with a space delimiter so we can put our JavaScript script in a readable way in that template and it willfinally  end on one line. Thus the best way to do this is tu put a JavaScipt file into an S3 buket and point to that bucket in the template. Since the script is small we choose to put everything in the template directly.<br />
Then declare Environment variables so we can reuse this function. At the end, we can see than we link the Lambda to a role. This role is pretty important, have a look:

```
  LambdaAutoApprovalServiceRole:
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
                  - codepipeline:GetPipeline
                  - codepipeline:GetPipelineState
                  - codepipeline:GetPipelineExecution
                  - codepipeline:ListPipelineExecutions
                  - codepipeline:ListPipelines
                  - codepipeline:PutApprovalResult
```

This role is about giving rights to write logs in CloudWatch Logs and validate Approvals within the CodePipeline pipelines. Note that if you have strict security policies, you shall restrict the accesses to the minimum in the `Resource` fields.

Now that the Lambda is created, we put the CloudWatch Event Rule definition:

```
  ScheduledRule:
    Type: AWS::Events::Rule
    Properties:
      Description: Scheduled CodePipeline approval
      ScheduleExpression: "cron(0 9 ? * MON *)"
      State: ENABLED
      Targets:
        -
          Arn: !GetAtt LambdaFunction.Arn
          Id: LambdaAutomaticApproval
```

This definition is short, it is just about the schedule (cron expression) and the target on which to link, our Lambda indeed.<br />

To make the CloudWatch Event Rule able to invoke the Lambda, we have to give it permission with a `AWS::Lambda::Permission` statement:
```
  PermissionForEventsToInvokeLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LambdaFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt ScheduledRule.Arn
```

So we are now good to go, with an automated deploiment on monday mornings!
