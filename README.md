# lab-week3-aws-cli

Group Members:
- Michael Flood
- Ray CHu
- Mellaad aktory

This repo contains scripts to automate AWS infrastructure setup using the AWS CLI.

---

## Script 1: Import SSH Key
Imports a locally generated public key as "bcitkey".

 `import-key.sh` 

```






```

[Import Key Pair Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/import-key-pair.html)

---

## Script 2: Create S3 Bucket
Creates an S3 bucket in `us-west-2`.

 `create-bucket.sh`  
```
#!/usr/bin/env bash

set -euo pipefail

# Check if the number of command-line arguments is correct
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <bucket_name>"
    exit 1
fi

# pass the bucket name as a positional parameter
bucket_name=$1

# Check if the bucket exists
if aws s3api head-bucket --bucket "$bucket_name" 2>/dev/null; then
    echo "Bucket $bucket_name already exists."
else
  aws s3api create-bucket --bucket "$bucket_name"
  echo "Bucket $bucket_name created."


```

[Create Bucket Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api/create-bucket.html)

---

## Script 3: Create VPC
Creates a new VPC and saves the VPC ID.

 `create-vpc.sh`  
```
#!/usr/bin/env bash

set -euo pipefail

region="us-west-2"
key_name="bcitkey"

source ./infrastructure_data

# Get most recent Debian AMI
debian_ami=$(aws ec2 describe-images \
  --owners "136693071363" \
  --filters 'Name=name,Values=debian-*-amd64-*' 'Name=architecture,Values=x86_64' 'Name=virtualization-type,Values=hvm' \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# Create security group allowing SSH and HTTP from anywhere
security_group_id=$(aws ec2 create-security-group --group-name MySecurityGroup \
 --description "Allow SSH and HTTP" --vpc-id $vpc_id --query 'GroupId' \
 --region $region \
 --output text)

aws ec2 authorize-security-group-ingress --group-id $security_group_id \
 --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $region

aws ec2 authorize-security-group-ingress --group-id $security_group_id \
 --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $region

# Launch an EC2 instance in the public subnet
instance_id=$(aws ec2 run-instances \
  --image-id "$debian_ami" \
  --count 1 \
  --instance-type t2.micro \
  --key-name "$key_name" \
  --security-group-ids "$security_group_id" \
  --subnet-id "$public_subnet_id" \
  --associate-public-ip-address \
  --region "$region" \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched EC2 instance: $instance_id"

# wait for ec2 instance to be running
aws ec2 wait instance-running --instance-ids "$instance_id" --region "$region"

# Get the public IP address of the EC2 instance
public_ip=$(aws ec2 describe-instances \
  --instance-ids "$instance_id" \
  --region "$region" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# Write instance data to a file
echo "Public IP: $public_ip"



```

[Create VPC Docs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-vpc.html)

---

## Script 4: Launch EC2 Instance
Creates subnet, launches EC2 instance, associates public IP, and saves data.

 `create-ec2.sh`  


```

#!/usr/bin/env bash
set -euo pipefail

# Variables
region="us-west-2"
vpc_cidr="10.0.0.0/16"
subnet_cidr="10.0.1.0/24"

# Create VPC
vpc_id=$(aws ec2 create-vpc \
  --cidr-block $vpc_cidr \
  --query 'Vpc.VpcId' \
  --output text \
  --region $region)

# Add Name tag to VPC
aws ec2 create-tags \
  --resources $vpc_id \
  --tags Key=Name,Value=MyVPC \
  --region $region

# Enable DNS hostnames in the VPC
aws ec2 modify-vpc-attribute \
  --vpc-id $vpc_id \
  --enable-dns-hostnames \
  --region $region \
  --no-cli-pager

# Create public subnet
subnet_id=$(aws ec2 create-subnet \
  --vpc-id $vpc_id \
  --cidr-block $subnet_cidr \
  --availability-zone ${region}a \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $region)

# Add Name tag to subnet
aws ec2 create-tags \
  --resources $subnet_id \
  --tags Key=Name,Value=PublicSubnet \
  --region $region

# Create Internet Gateway
igw_id=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text \
  --region $region)

# Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway \
  --vpc-id $vpc_id \
  --internet-gateway-id $igw_id \
  --region $region

# Create Route Table for the VPC
route_table_id=$(aws ec2 create-route-table \
  --vpc-id $vpc_id \
  --query 'RouteTable.RouteTableId' \
  --output text \
  --region $region)

# Associate Route Table with the public subnet
aws ec2 associate-route-table \
  --subnet-id $subnet_id \
  --route-table-id $route_table_id \
  --region $region

# Create route to the Internet Gateway for 0.0.0.0/0
aws ec2 create-route \
  --route-table-id $route_table_id \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $igw_id \
  --region $region

# Write infrastructure data to a file for later use
cat > infrastructure_data << EOF
vpc_id=${vpc_id}
subnet_id=${subnet_id}
igw_id=${igw_id}
route_table_id=${route_table_id}
EOF

echo "Infrastructure setup complete. Data saved to infrastructure_data"


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
