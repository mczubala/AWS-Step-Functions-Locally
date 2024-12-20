# AWS Step Functions Locally

This guide explains how to set up and run AWS Step Functions locally using AWS SAM (Serverless Application Model) and Docker.

---

## Prerequisites

- Install **AWS SAM CLI** ([Installation Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)).
- Docker installed and running on your system.

---

## Steps to Run AWS Step Functions Locally

### 1. Start the Lambda Function Locally

Run the following command to start the Lambda function locally:
```bash
sam local start-lambda
```

If you need to specify a specific AWS credentials profile, use:
```bash
sam local start-lambda --profile <YOUR_PROFILE>
```

### 2. Create the `template.yaml`

The `template.yaml` file is required by AWS SAM to define your serverless resources. Below is an example configuration:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  InitiateUploadLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_InitiateUpload_Generated::InitiateUpload
      Runtime: dotnet8
      CodeUri: <PATH_TO_YOUR_DLL>
      Timeout: 30
  
  CalculateByteRangesLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_CalculateByteRanges_Generated::CalculateByteRanges
      Runtime: dotnet8
      CodeUri: <PATH_TO_YOUR_DLL>
      Timeout: 30

  UploadPartLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_UploadPart_Generated::UploadPart
      Runtime: dotnet8
      CodeUri: <PATH_TO_YOUR_DLL>
      Timeout: 30

  CompleteUploadLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: zmph-sys-xperi.Functions::zmph_sys_xperi.Functions.Functions_CompleteUpload_Generated::CompleteUpload
      Runtime: dotnet8
      CodeUri: <PATH_TO_YOUR_DLL>
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
      RoleArn: <YOUR_ROLE_ARN>
```

Replace `<PATH_TO_YOUR_DLL>` and `<YOUR_ROLE_ARN>` with your respective paths and role ARN.

---

### 3. Run Step Functions Locally

Start AWS Step Functions locally using Docker:
```bash
docker run -p 8083:8083 --env-file aws-stepfunctions-local-credentials.txt amazon/aws-stepfunctions-local
```

#### Create `aws-stepfunctions-local-credentials.txt`
This file contains credentials and configuration for running Step Functions locally. Example:
```text
AWS_DEFAULT_REGION=us-east-1
LAMBDA_ENDPOINT=http://host.docker.internal:3001
AWS_ACCESS_KEY_ID=AKIATP3BOXG36USHJEPQ
AWS_SECRET_ACCESS_KEY=0aAjcOmsMDL8eHElLDdLNp2gaQxrcn4JvByrwJQl
```

---

### 4. Create a State Machine

Run the following command to create a Step Functions state machine locally:
```bash
aws stepfunctions create-state-machine \
  --definition file://state-machine.json \
  --name "MultipartUploadStateMachine" \
  --role-arn "arn:aws:iam::123456789012:role/DummyRole" \
  --endpoint-url http://localhost:8083
```

#### Example `state-machine.json`
```json
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
```

---

### 5. Start the State Machine

To execute the state machine, use:
```bash
aws stepfunctions start-execution \
  --state-machine-arn "arn:aws:states:us-east-1:123456789012:stateMachine:MultipartUploadStateMachine" \
  --endpoint-url http://localhost:8083
```

