# lab-week3-aws-cli

Group Members:
- Your Name
- Partner Name

This repo contains scripts to automate AWS infrastructure setup using the AWS CLI.

---

## Script 1: Import SSH Key
Imports a locally generated public key as "bcitkey".

 `import-key.sh` 

```
#!/bin/bash

aws ec2 import-key-pair \
    --key-name "bcitkey" \
    --public-key-material fileb://~/.ssh/bcitkey.pub \
    --region us-west-2





```

[Import Key Pair Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/import-key-pair.html)

---

## Script 2: Create S3 Bucket
Creates an S3 bucket in `us-west-2`.

 `create-bucket.sh`  
```
#!/bin/bash


BUCKET_NAME="bcit-cli-bucket-$(date +%s)"

aws s3api create-bucket \
    --bucket "$BUCKET_NAME" \
    --region us-west-2 \
    --create-bucket-configuration LocationConstraint=us-west-2

echo "Created bucket: $BUCKET_NAME"


```

[Create Bucket Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api/create-bucket.html)

---

## Script 3: Create VPC
Creates a new VPC and saves the VPC ID.

 `create-vpc.sh`  
```

#!/bin/bash

VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --query 'Vpc.VpcId' \
    --output text \
    --region us-west-2)

echo "Created VPC: $VPC_ID"

aws ec2 create-tags \
    --resources "$VPC_ID" \
    --tags Key=Name,Value=bcit-vpc \
    --region us-west-2

echo "$VPC_ID" > vpc_id.txt


```

[Create VPC Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-vpc.html)

---

## Script 4: Launch EC2 Instance
Creates subnet, launches EC2 instance, associates public IP, and saves data.

 `create-ec2.sh`  


```

#!/bin/bash

KEY_NAME="bcitkey"
AMI_ID="ami-xxxxxxxxxxxxxxxxx"   # Replace with valid Debian AMI for us-west-2
SECURITY_GROUP_ID="sg-xxxxxxxxxxxxxxx"  # Replace with your security group ID
VPC_ID=$(cat vpc_id.txt)

SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block 10.0.1.0/24 \
    --query 'Subnet.SubnetId' \
    --output text \
    --region us-west-2)

echo "Created Subnet: $SUBNET_ID"

INSTANCE_ID=$(aws ec2 run-instances \
    --image-id "$AMI_ID" \
    --count 1 \
    --instance-type t2.micro \
    --key-name "$KEY_NAME" \
    --security-group-ids "$SECURITY_GROUP_ID" \
    --subnet-id "$SUBNET_ID" \
    --associate-public-ip-address \
    --query 'Instances[0].InstanceId' \
    --output text \
    --region us-west-2)

echo "Launched Instance: $INSTANCE_ID"

aws ec2 wait instance-running \
    --instance-ids "$INSTANCE_ID" \
    --region us-west-2


PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text \
    --region us-west-2)

echo "Public IP: $PUBLIC_IP"

echo "Instance ID: $INSTANCE_ID" > instance_data
echo "Public IP: $PUBLIC_IP" >> instance_data



```

 [Run Instances Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/run-instances.html)  
 [Describe Instances Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html)

---

## Notes

Make sure to:
- Replace placeholder values for AMI ID and Security Group ID.
- Run these scripts in order.
- SSH into your instance using:
```bash
ssh -i ~/.ssh/bcitkey ec2-user@<PUBLIC_IP>
