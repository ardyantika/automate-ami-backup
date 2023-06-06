# automate-ami-backup
Automate AMI creation using ECS Lifecyle Management + Lambda + S3
<br><br>
## High-level Steps
1. [Create a dedicated S3 bucket to store AMI, make sure to enable EventBridge notification in the bucket](https://github.com/ardyantika/automate-ami-backup/blob/main/README.md#part-1-s3-preparation)
2. Create a new lifecyle policy in EBS to automate AMI creation
3. Create a new policy and role for Lambda
4. Develop 2 Lambda functions for: 1) migrate/store AMI to S3 and 2) cleanup AMI + related snapshot (once AMI successfully stored in S3)
5. New EventBridge Rules as a trigger to Lambda to automate storing AMI to S3 and cleanup AMI (including the related snapshots)

<br><br>
## Part 1. S3 Preparation
This part should be straight forward, all best practices related to bucket creation are implicitly applied, such as: enable encryption, set bucket and objects are not public.

1. Create new dedicated S3 bucket
2. To automate cleanup AMI process once AMI successfully stored in S3, we need to enable bucket notification to AWS EventBridge
<img width="1119" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/a8e9830d-4fd8-411a-9744-23d28a5c018c">
3. Later, we will create new rule in EventBridge to make use of this notification to trigger a Lambda function for AMI cleanup process.

<br><br>
## Part 2. EBS Lifecycle Management
1. Got to EC2 management console and under Elastic Block Store sub-menu, choose "Lifecycle Manager"
<img width="1472" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/0d27b87c-36fa-4c6e-87d0-865d8cbb1496">

2. Click on "Create lifecyle policy" button
3. Choose "EBS-backed AMI policy" as policy type and click on "Next" button
<img width="811" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/ce70be74-5581-4676-947c-8e5aca3eae59">

4. Configure tag to identify resources that would be marked as part of this policy
<img width="1103" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/ee19a88d-a525-4b1e-8f93-c5b961143de7">

5. Configure new schedule for weekly AMI backup, adjust configurations accordingly, and click on "Review policy" button
<img width="1109" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/35c14aaf-bf04-446e-b4a9-9de25b11f96b">

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
<img width="1438" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/f6ccd657-1773-489b-8cef-fc942a70cb03">

4. Click "Next" button, search newly created policy "PolicyForLambdaBackupAMI"
5. Select the policy and click on next button. Put an appropriate name (For Example: "LambdaRoleBackupAMI") and click "Create" button
<img width="1467" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/ce8f8de1-3834-4f3c-84a4-8b2743569e41">

<br><br>
## Part 4. Lambda Functions
Make sure to configure runtime to use Python 3.10 on x86_64 and choose execution role to use the newly created role in Part 3
<img width="1101" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/7dde8f58-a3be-4883-a610-1776d6fb41bb">

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
<img width="1442" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/0d62e729-9ffe-4db4-aa19-48876e1f8db5">

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
<img width="819" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/f73e17d9-e622-4a67-8442-da4726c200f0">

2. Click "Next" button and select target as Lambda function
<img width="1134" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/07ff4c9a-a8a0-4dde-9135-38a4b4aa693c">
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
<img width="819" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/7eb5bc41-45b1-4d81-892e-2cd76c5fc4c5">

2. Click "Next" button and select target as Lambda function
<img width="1105" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/991c3bef-b9e9-4ba9-8c3e-df4cb8aa4a17">
3. Click "Next", then Review and click on "Create rule"
