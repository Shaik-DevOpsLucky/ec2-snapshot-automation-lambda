# ec2-snapshot-automation-lambda

# Lambda Function to Take a Snapshot of an EC2 Instance

This guide explains how to create a Lambda function that takes snapshots of EBS volumes attached to an EC2 instance, using AWS Lambda and Amazon EventBridge.

---

## **Prerequisites**
- AWS account with permissions to manage EC2, Lambda, and IAM.
- Instance IDs for the EC2 machines you want to back up.
- Time zone for EventBridge rules must be **UTC**.

---

## **Task 1: Create EC2 Instance**
1. Create one Windows or Linux EC2 instance in the AWS Management Console.

---

## **Task 2: Create IAM Role for Lambda**
We will create a role `Ec2_Lambda_Snapshot` with permissions to create snapshots of the EC2 instance's EBS volumes.

### **Step 1 – Create IAM Role**
1. Create a role named **`Ec2_Lambda_Snapshot`**.

### **Step 2 – Create IAM Policy**
**Name:** `Lambda_Snapshot_Backup`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateSnapshots",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSnapshot",
                "ec2:DescribeVolumes",
                "ec2:DescribeInstances"
            ],
            "Resource": "arn:aws:ec2:ap-southeast-1:707728974508:instance/i-0bd1ca1753e7d383f"
        }
    ]
}
```

> **Note:** Replace the `instance ARN` with your own EC2 instance ARN.

3. Attach this policy to the **`Ec2_Lambda_Snapshot`** role.

---

## **Task 3: Create Lambda Function (Snapshot)**

### **Lambda Function – Take Snapshot**
1. Create a new Lambda function named **`DB1_Server_Snapshot`**.
2. Runtime: **Python 3.11**.
3. Permissions: Assign the **`Ec2_Lambda_Snapshot`** role.
4. Add the following code:

```python
import boto3
from datetime import datetime

region = 'ap-southeast-1'
instances = ['i-0e19d1db3f8e3782c']
ec2_resource = boto3.resource('ec2', region_name=region)

def lambda_handler(event, context):
    # Take snapshots of the EBS volumes
    for instance_id in instances:
        instance = ec2_resource.Instance(instance_id)
        for volume in instance.volumes.all():
            snapshot = volume.create_snapshot(
                Description=f"Snapshot of {instance_id} taken at {datetime.now().isoformat()}"
            )
            print(f'Snapshot {snapshot.snapshot_id} created for volume {volume.volume_id} of instance {instance_id}')
            
    print('Snapshots taken for all volumes attached to the instances.')

# Test the function locally (if needed)
if __name__ == "__main__":
    lambda_handler(None, None)
```

> **Note:**
> - Change `region` and `instances` values as required.
> - Multiple instances can be listed as:
>   ```python
>   instances = ['i-03ed8bb3a2e2ffb47', 'i-061e8e8245a3f1e15']
>   ```

5. Deploy and test the function to verify snapshots are created.
6. Save.

---

## **Task 4: Create EventBridge Rule to Trigger Lambda**
We will trigger the Lambda function using **EventBridge** rules.

---

### **Rule Creation**
1. Open the Lambda function **`DB1_Server_Snapshot`**.
2. Click **Add Trigger** → **EventBridge**.
3. Create a new rule:
   - **Name:** `DB1_Server_Snapshot`
   - **Description:** `DB1_Server_Snapshot at 10 PM MYT`
   - **Rule type:** Schedule expression
   - **Cron expression (UTC):**
     ```text
     cron(0 14 ? * * *)
     ```
   - This runs at **10:00 PM MYT = 2:00 PM UTC**.
4. Target: Lambda function.
5. Save.

> **Note:** EventBridge cron expressions must be set in **UTC**.

---

## **Summary**
- IAM Role `Ec2_Lambda_Snapshot` allows Lambda to create snapshots of EBS volumes.
- Lambda function takes a snapshot of all volumes attached to a given EC2 instance.
- EventBridge rules trigger the Lambda function on a set schedule.
