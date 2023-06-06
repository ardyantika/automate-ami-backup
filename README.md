# automate-ami-backup
Automate AMI creation using ECS Lifecyle Management + Lambda + S3
<br><br>
## High-level Steps
1. [Create a dedicated S3 bucket to store AMI, make sure to enable EventBridge notification in the bucket](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-1-s3-preparation)
2. [Create a new lifecyle policy in EBS to automate AMI creation](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-2-ebs-lifecycle-management)
3. [Create a new policy and role for Lambda](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-3-create-policy-and-role-for-lambda)
4. [Develop 2 Lambda functions for: 1) migrate/store AMI to S3 and 2) cleanup AMI + related snapshot (once AMI successfully stored in S3)](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-4-lambda-functions)
5. [New EventBridge Rules as a trigger to Lambda to automate storing AMI to S3 and cleanup AMI (including the related snapshots)](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-5-eventbridge-rules)

<br><br>
## Part 1. S3 Preparation
This part should be straight forward, all best practices related to bucket creation are implicitly applied, such as: enable encryption, set bucket and objects are not public.

1. Create new dedicated S3 bucket
2. To automate cleanup AMI process once AMI successfully stored in S3, we need to enable bucket notification to AWS EventBridge (Under "bucket properties" menu)
<img width="1128" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/2aea8c56-2ff1-4bda-b5e0-78e6cdaa3d1e">
3. Later, we will create new rule in EventBridge to make use of this notification to trigger a Lambda function for AMI cleanup process.

<br><br>
## Part 2. EBS Lifecycle Management
1. Go to EC2 management console and under Elastic Block Store sub-menu, choose "Lifecycle Manager"
<img width="1470" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/1b03e04e-dfe8-41d9-91aa-d4fbf90ecc0f">


2. Click on "Create lifecyle policy" button
3. Choose "EBS-backed AMI policy" as policy type and click on "Next" button
<img width="824" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/11b2b7bf-937d-4a2b-b300-5df411e12b91">

4. Configure tag to identify resources that would be marked as part of this policy
<img width="1105" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/77685621-3b5d-4cda-b82c-bc8db183a041">

5. Configure new schedule for weekly AMI backup, adjust configurations accordingly, and click on "Review policy" button
<img width="1128" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/93885963-39f9-4905-81ce-5b31ca7d10b9">

6. Once review is completed, click on "Create policy" button and the policy will be created and enabled.

<br><br>
## Part 3. Create Policy and Role for Lambda
### Create Policy 
1. Go to IAM - Access management - Policies
2. Create new policies "PolicyForLambdaBackupAMI" using following permission:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ebs:CompleteSnapshot",
                "ebs:GetSnapshotBlock",
                "ebs:ListChangedBlocks",
                "ebs:ListSnapshotBlocks",
                "ebs:PutSnapshotBlock",
                "ebs:StartSnapshot",
                "ec2:CreateImage",
                "ec2:CreateRestoreImageTask",
                "ec2:CreateStoreImageTask",
                "ec2:CreateTags",
                "ec2:DeleteSnapshot",
                "ec2:DeregisterImage",
                "ec2:DescribeImageAttribute",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeStoreImageTasks",
                "ec2:DescribeTags",
                "ec2:GetEbsEncryptionByDefault",
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObject",
                "ssm:GetParameter",
                "ssm:GetParametersByPath"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

### Create Roles
1. Go to IAM - Access management - Roles
2. Click "Create role" button
3. Choose "Custom trust policy" as Trusted entity type and use following JSON script
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
<img width="1353" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/4604fef7-f688-43ad-a45f-960fd44a29bf">

4. Click "Next" button, search newly created policy "PolicyForLambdaBackupAMI"
5. Select the policy and click on next button. Put an appropriate name (For Example: "LambdaRoleBackupAMI") and click "Create" button
<img width="1397" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/90b1b68f-77be-4f20-9b2d-47a478630e9e">

<br><br>
## Part 4. Lambda Functions
Make sure to configure runtime to use Python 3.10 on x86_64 and choose execution role to use the newly created role in Part 3
<img width="1089" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/c69478b4-f31a-4bbf-aa06-a779d696a031">

### Store AMI to S3
Code:
```
import json
import boto3
import os
import logging
from botocore.exceptions import ClientError
from botocore.config import Config

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)
client = boto3.client('ec2')
    
def lambda_handler(event, context):
  logger.info("Received event: %s", event)

  ImageId = event["detail"]["ImageId"]
  BucketName = os.environ["TARGET_BACKUP_BUCKET"]

  logger.info("Image ID: %s", ImageId)
  logger.info("Bucket Name: %s", BucketName)
    
  # Create Store AMI task to S3
  try:
    response = client.create_store_image_task(
      ImageId=ImageId,
      Bucket=BucketName
    );
    ObjectKey = response["ObjectKey"]

    return {
      "ImageId": ImageId,
      "BucketName": BucketName,
      "ObjectKey": ObjectKey
    }
  except ClientError as err:
    logger.exception("Couldn't create store AMI task for: %s", ImageId)
    
    raise
    return {
      "Status": "error",
      "ImageId": ImageId,
      "BucketName": BucketName
    }
```

### Cleanup AMI
Code:
```
import json
import boto3
import os
import logging
from botocore.exceptions import ClientError
from botocore.config import Config

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)
client = boto3.client('ec2')

def lambda_handler(event, context):
  logger.info("Received event: %s", event)

  ImageId = event["detail"]["object"]["key"].replace('.bin','')
  logger.info("AMI ID: %s", ImageId)
  snapshot_list = []
    
  # Check AMI details
  try:
    response = client.describe_images(ImageIds=[ImageId]);
    if len(response["Images"]) == 0:
      return {
        "Status": "imageNotFound",
        "ImageId": ImageId
      }
    Status = response["Images"][0]["State"]
    Snapshots = response["Images"][0]["BlockDeviceMappings"]
    logger.info("AMI Status: %s", Status)
    for snapshot in Snapshots:
      snapshot_list.append(snapshot["Ebs"]["SnapshotId"])
    logger.info("AMI Snapshots: %s", snapshot_list)
  except ClientError as err:
    logger.exception("Couldn't get status for AMI: %s", ImageId)
    raise
    return {
      "Status": "error",
      "ImageId": ImageId
    }
  
  # Deregister AMI
  try:
    response = client.deregister_image(ImageId=ImageId)
    logger.info("AMI %s has been deregistered", ImageId)
  except ClientError as err:
    logger.exception("Failed to deregister AMI: %s", ImageId)
    raise
    return {
      "Status": "error",
      "ImageId": ImageId
    }

  # Delete associated snapshots
  for i in snapshot_list:
    try:
      response = client.delete_snapshot(SnapshotId=i)
      logger.info("SnapshotId %s from AMI %s has been deleted", i, ImageId)
    except ClientError as err:
      logger.exception("Couldn't delete snapshotId: %s", i)
      raise
      return {
        "Status": "error",
        "SnapshotId": i
      }
  logger.info("AMI %s has been deleted completely", ImageId)
  
  return {
    "Status": "Deleted",
    "ImageId": ImageId
  }

```

<br><br>
## Part 5. EventBridge Rules
Create following rules in EventBridge console (Under "Buses" menu):
<img width="1442" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/a1b76127-7fb5-48f7-b31c-034e7cb0a0d1">

### Rule 1: Trigger Store AMI to S3
1. Create new rules with following event pattern:
```
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 AMI State Change"],
  "detail": {
    "State": ["available"]
  }
}
```
<img width="838" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/597ccc7c-3320-4890-82b6-6a1fa0ed287f">

2. Click "Next" button and select target as Lambda function
<img width="1108" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/94f4a8d5-4437-45b5-b0b5-99425db57109">

3. Click "Next", then Review and click on "Create rule"

### Rule 2: Trigger Cleanup AMI
1. Create new rules with following event pattern:
```
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["YOUR BACKUP BUCKET"]
    }
  }
}
```
<img width="810" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/176cf6d7-3371-4926-815f-3dfd4c97cecc">

2. Click "Next" button and select target as Lambda function
<img width="1095" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/6367df03-5fc0-439f-874e-a25814c03299">

3. Click "Next", then Review and click on "Create rule"
