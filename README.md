# AWS-Step-Functions-Locally
Download AWS SAM 
sam local start-lambda 
optional: sam local start-lambda --profile XXXX   (if need to start with specific AWS credentials profile)
necessary template.yaml
Handler can be found when using Mock Test Lambda Tool in serverless.template
CodeUri is a directory where lambda functions are compiled into .dll
e.g. 



AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  InitiateUploadLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_InitiateUpload_Generated::InitiateUpload
      Runtime: dotnet8
      CodeUri: C:/Users/MarcinCzubala/RiderProjects/zmph-sys-xperi/src/zmph-sys-xperi.Functions/bin/Debug/net8.0
      Timeout: 30
      
  CalculateByteRangesLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_CalculateByteRanges_Generated::CalculateByteRanges
      Runtime: dotnet8
      CodeUri: C:/Users/MarcinCzubala/RiderProjects/zmph-sys-xperi/src/zmph-sys-xperi.Functions/bin/Debug/net8.0
      Timeout: 30
          
  UploadPartLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_UploadPart_Generated::UploadPart
      Runtime: dotnet8
      CodeUri: C:/Users/MarcinCzubala/RiderProjects/zmph-sys-xperi/src/zmph-sys-xperi.Functions/bin/Debug/net8.0
      Timeout: 30
          
  CompleteUploadLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_CompleteUpload_Generated::CompleteUpload
      Runtime: dotnet8
      CodeUri: C:/Users/MarcinCzubala/RiderProjects/zmph-sys-xperi/src/zmph-sys-xperi.Functions/bin/Debug/net8.0
      Timeout: 30
          
  MultipartUploadStateMachine:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      DefinitionString: !Sub |
        {
          "Comment": "Step Function for multipart upload",
          "StartAt": "InitiateUpload",
          "States": {
            "InitiateUpload": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:InitiateUploadLambda",
              "ResultPath": "$.initiateResponse",
              "Next": "CalculateByteRanges"
            },
            "CalculateByteRanges": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CalculateByteRangesLambda",
              "ResultPath": "$.byteRanges",
              "Next": "UploadParts"
            },
            "UploadParts": {
              "Type": "Map",
              "ItemsPath": "$.byteRanges",
              "Iterator": {
                "StartAt": "UploadPart",
                "States": {
                  "UploadPart": {
                    "Type": "Task",
                    "Resource": "arn:aws:lambda:us-east-1:123456789012:function:UploadPartLambda",
                    "ResultPath": "$.partETag",
                    "End": true
                  }
                }
              },
              "ResultPath": "$.uploadedParts",
              "Next": "CompleteUpload"
            },
            "CompleteUpload": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CompleteUploadLambda",
              "Parameters": {
                "Bucket.$": "$.initiateResponse.Bucket",
                "Key.$": "$.initiateResponse.Key",
                "UploadId.$": "$.initiateResponse.UploadId",
                "Parts.$": "$.uploadedParts"
              },
              "End": true
            }
          }
        }
      RoleArn: arn:aws:iam::960515033542:role/sofomo_lukasz
      


Step functions local 
docker run -p 8083:8083 --env-file aws-stepfunctions-local-credentials.txt amazon/aws-stepfunctions-local

need to create aws-stepfunctions-local-credentials.txt
e.g.
AWS_DEFAULT_REGION=us-east-1
LAMBDA_ENDPOINT=http://host.docker.internal:3001
AWS_ACCESS_KEY_ID=AKIATP3BOXG36USHJEPQ"
AWS_SECRET_ACCESS_KEY="0aAjcOmsMDL8eHElLDdLNp2gaQxrcn4JvByrwJQl"

create step function
aws stepfunctions create-state-machine --definition file://state-machine.json --name "MultipartUploadStateMachine" --role-arn "arn:aws:iam::123456789012:role/DummyRole" --endpoint-url http://localhost:8083

state-machine.json example:
{
  "Comment": "Step Function for multipart upload",
  "StartAt": "InitiateUpload",
  "States": {
    "InitiateUpload": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:InitiateUploadLambda",
      "ResultPath": "$.initiateResponse",
      "Next": "CalculateByteRanges"
    },
    "CalculateByteRanges": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CalculateByteRangesLambda",
      "ResultPath": "$.byteRanges",
      "Next": "UploadParts"
    },
    "UploadParts": {
      "Type": "Map",
      "ItemsPath": "$.byteRanges",
	  "Parameters": {
    "UploadId.$": "$.initiateResponse.UploadId",
    "Bucket.$": "$.initiateResponse.BucketName",
    "Key.$": "$.initiateResponse.Key",
    "PartNumber.$": "$$.Map.Item.Value.PartNumber",
    "Start.$": "$$.Map.Item.Value.Start",
    "End.$": "$$.Map.Item.Value.End"
  },
      "Iterator": {
        "StartAt": "UploadPart",
        "States": {
          "UploadPart": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:UploadPartLambda",
            "ResultPath": "$.partETag",
            "End": true
          }
        }
      },
      "ResultPath": "$.uploadedParts",
      "Next": "CompleteUpload"
    },
    "CompleteUpload": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CompleteUploadLambda",
      "Parameters": {
        "Bucket.$": "$.initiateResponse.BucketName",
        "Key.$": "$.initiateResponse.Key",
        "UploadId.$": "$.initiateResponse.UploadId",
        "Parts.$": "$.uploadedParts"
      },
      "End": true
    }
  }
}


start step function
aws stepfunctions start-execution --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:MultipartUploadStateMachine" --endpoint-url http://localhost:8083

