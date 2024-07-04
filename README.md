# AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM
we'll create a Lambda function that identifies EBS snapshots that are no longer associated with any active EC2 instance and deletes them to save on storage costs.
For startups and mid-scale organizations, setting up a traditional data center involves significant overhead. It requires purchasing and arranging servers, managing a dedicated team of system administrators, and continuously monitoring the infrastructure to prevent issues like latency, data loss, or server downtime. This entire process can be both costly and resource-intensive.

Fortunately, AWS provides a range of services that can help streamline infrastructure management and reduce costs. One such service is Amazon Elastic Block Store (EBS), which offers persistent block storage for Amazon EC2 instances. However, if not managed properly, EBS snapshots and volumes can gather and lead to unnecessary expenses.

Understanding EBS Snapshots

EBS snapshots are incremental backups of your EBS volumes. They are stored in Amazon S3 and can be used to create new volumes or restore existing ones. AWS charges for these snapshots even if the associated EC2 instance is deleted. Therefore, it’s crucial to identify and remove stale snapshots to optimize costs.

Implementing Cost Optimization with AWS Lambda

AWS Lambda is a serverless compute service that allows you to run code without provisioning or managing servers. By leveraging AWS Lambda, you can automate the cleanup of stale EBS snapshots. Here’s a step-by-step guide on how to achieve this:

Step 1: Set Up Your Environment

Create an EC2 Instance and Volume:

Log into your AWS account.

Navigate to the EC2 dashboard and create an instance named test-ec2.

Create a key pair named aws-login for accessing your instance.

![Screenshot 2024-07-04 155714](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/38d270de-082b-43bb-b4ba-7ef3be479567)


![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/d4b826fe-3335-4036-a486-6e2f62dcc09a)


Once the instance is running, create an EBS volume and attach it to your EC2 instance or you can use the default volume which gets created automatically with EC2 instance.

Create an EBS Snapshot:

Go to the EBS section in the AWS Management Console.

Select the volume attached to your test-ec2 instance and create a snapshot.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/a797c81a-de37-44b8-bf97-d255d48fdcf3)


Step 2: Configure AWS Lambda

Create a Lambda Function:

Navigate to the AWS Lambda console and create a function named cost-optimization-EBS-snapshot.

Select Python 3.10 as the runtime environment.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/9320624c-cb5a-4985-8117-ea8712a7b590)
![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/c8626eb5-3802-446a-8880-bcdb834d68e2)


Write and Deploy the Code:

Copy the Python code from your preferred repository (e.g., GitHub).

In the Lambda console, paste the code into the function editor, save it, and deploy the function.

The code I am using here is:

Automating the Cleanup of Stale EBS Snapshots Using AWS Lambda


import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    deleted_snapshots = []

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])
    print(f"Found {len(response['Snapshots'])} snapshots.")

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])
    print(f"Found {len(active_instance_ids)} active instances.")

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')
        print(f"Checking snapshot {snapshot_id} with volume {volume_id}")

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            deleted_snapshots.append(snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    deleted_snapshots.append(snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    deleted_snapshots.append(snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
                else:
                    print(f"Error checking volume {volume_id} for snapshot {snapshot_id}: {e}")

    print("Lambda function execution completed.")
    return {"deleted_snapshots": deleted_snapshots}


Explanation of the code: This Python script is designed to run as an AWS Lambda function. It automates the cleanup of stale Amazon Elastic Block Store (EBS) snapshots to optimize costs. The script uses the Boto3 library to interact with AWS services. It retrieves all EBS snapshots owned by the account, identifies active EC2 instances, and deletes snapshots that are not attached to any volume or are associated with volumes that are not attached to running instances. This helps in reducing unnecessary storage costs by removing unused snapshots.

Output after deploying and testing:

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/a7d2eb9e-2e85-448f-a08d-bbf09f15c380)


The logs indicate that the Lambda function executed successfully, identified one snapshot and one active instance, and did not delete any snapshots. The snapshot snap-0e79ed6135368a22e was associated with the volume vol-0d0449f2d3e39091f.

Analyzing Why Snapshots Were Not Deleted

Here are a few possible reasons why the snapshot was not deleted:

The volume vol-0d0449f2d3e39091f is attached to a running instance.

There might be an error in how snapshots are being checked or the criteria for deletion.

To provide further clarity, we can add more logging to check the attachment status of the volume.

Updated Version of the Lambda Function with Additional Logging

The updated version of the Lambda function includes more detailed logging to provide insights into why certain snapshots are not deleted.

If you want to delete snapshots based on different criteria, you can modify the logic accordingly. For example, you might want to delete snapshots older than a certain number of days, or snapshots that are not associated with any volumes at all.

Here is how you can modify the function to delete snapshots older than a certain number of days (e.g., 30 days):


```python
```markdown
import boto3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    deleted_snapshots = []
    days_old = 30  # Change this to the number of days you want

    # Calculate the cutoff date as an offset-naive datetime
    cutoff_date = datetime.utcnow() - timedelta(days=days_old)

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])
    print(f"Found {len(response['Snapshots'])} snapshots.")

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])
    print(f"Found {len(active_instance_ids)} active instances.")

    # Iterate through each snapshot and delete if it's older than the cutoff date and not attached to a volume
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')
        start_time = snapshot['StartTime']

        # Convert start_time to an offset-naive datetime for comparison
        start_time_naive = start_time.replace(tzinfo=None)

        if start_time_naive < cutoff_date:
            print(f"Snapshot {snapshot_id} is older than {days_old} days.")

            if not volume_id:
                # Delete the snapshot if it's not attached to any volume
                ec2.delete_snapshot(SnapshotId=snapshot_id)
                deleted_snapshots.append(snapshot_id)
                print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
            else:
                # Check if the volume still exists
                try:
                    volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                    if not volume_response['Volumes'][0]['Attachments']:
                        ec2.delete_snapshot(SnapshotId=snapshot_id)
                        deleted_snapshots.append(snapshot_id)
                        print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
                except ec2.exceptions.ClientError as e:
                    if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                        # The volume associated with the snapshot is not found (it might have been deleted)
                        ec2.delete_snapshot(SnapshotId=snapshot_id)
                        deleted_snapshots.append(snapshot_id)
                        print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
                    else:
                        print(f"Error checking volume {volume_id} for snapshot {snapshot_id}: {e}")
        else:
            print(f"Snapshot {snapshot_id} is newer than {days_old} days and will not be deleted.")

    print("Lambda function execution completed.")
    return {"deleted_snapshots": deleted_snapshots}
```

This version includes a condition to delete snapshots older than 30 days. You can adjust the days_old variable to specify the age of snapshots you want to target for deletion. Deploy and run this version to see if it meets your requirements.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/0b5da1a6-7d65-4e62-a6bc-4ec026d47406)


The logs indicate that the Lambda function executed successfully without deleting any snapshots because the snapshot snap-0e79ed6135368a22e is newer than 30 days.

Test the Lambda Function:

Configure a test event in the Lambda console.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/51605d00-c118-4a74-b101-c6fabd035f24)


Increase the default execution time to 10 seconds under the configuration tab.

Save and manually trigger the test event to ensure the function works correctly.

Step 3: Set Up IAM Policies

Create a Custom IAM Policy:

Go to the IAM console and create a new policy.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/ba583433-157b-4b84-a9c2-31333e1904f4)


Select the EC2 service and add permissions for listing, describing, and deleting snapshots, volumes, and instances.

Name the policy cost-optimization-ebs and create it.

Attach the Policy to the Lambda Role:

In the Lambda function configuration, navigate to the permissions tab.

Click on the role name, go to permissions, and attach the cost-optimization-ebs policy.

![image](https://github.com/shivanii1702/AWS-Cost-Optimization-Automating-EBS-Cleanup-with-Lambda-and-IAM/assets/91375420/f12ea6f6-e1fa-423d-858c-608867bb13e2)


Test the Cleanup Process:

Run the Lambda function again to ensure that it correctly identifies and deletes stale snapshots while leaving active ones intact.

Conclusion:-

By automating the cleanup of stale EBS snapshots using AWS Lambda and IAM policies, you can significantly reduce unnecessary costs and improve resource management. This approach not only saves money but also ensures that your infrastructure remains lean and efficient, allowing your organization to focus on core business activities.
