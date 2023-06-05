# automate-ami-backup
Automate AMI creation using ECS Lifecyle Management + Lambda + S3

## High-level Components
1. Dedicated S3 bucket to store AMI, make sure to enable EventBridge notification in the bucket
2. New lifecyle policy in EBS to automate AMI creation
3. New EventBridge Rules as a trigger to Lambda to automate storing AMI to S3 and cleanup AMI (including the related snapshots)
4. Lambda function to migrate/store AMI to S3
5. Lambda function to cleanup AMI + related snapshot, once AMI successfully stored in S3

## Part 1. S3 Preparation
This part should be straight forward, all best practices related to bucket creation are implicitly applied, such as: enable encryption, set bucket and objects are not public.

1. Create new dedicated S3 bucket
2. To automate cleanup AMI process once AMI successfully stored in S3, we need to enable bucket notification to AWS EventBridge
<img width="1119" alt="image" src="https://github.com/ardyantika/automate-ami-backup/assets/4241098/a8e9830d-4fd8-411a-9744-23d28a5c018c">
3. Later, we will create new rule in EventBridge to make use of this notification to trigger a Lambda function for AMI cleanup process.

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

## Part 3. EventBridge Rules
### Rule 1: Trigger Store AMI to S3
Event pattern:
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 AMI State Change"],
  "detail": {
    "State": ["available"]
  }
}

### Rule 2: Trigger Cleanup AMI

## Part 4. Lambda Function - Store AMI to S3
## Part 5. Lambda Function - Cleanup AMI
