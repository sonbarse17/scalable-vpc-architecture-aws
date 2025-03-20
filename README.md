# Deploy Scalable VPC Architecture on AWS Cloud

![AWS-Cloud](https://imgur.com/AXD50yl.png)

### TABLE OF CONTENTS

1. [Goal](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/README.md#goal)
2. [Pre-Requisites](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/README.md#pre-requisites)
3. [Pre-Deployment](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/README.md#pre-deployment)
4. [VPC Deployment](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/README.md#vpc-deployment)
5. [Validation](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/README.md#validation)

## Goal

Deploy a Modular and Scalable Virtual Network Architecture with Amazon VPC.

## Pre-Requisites

1. You must be having an [AWS account](https://aws.amazon.com/) to create infrastructure resources on AWS cloud.
2. [Source Code](https://github.com/NotHarshhaa/DevOps-Projects/blob/master/DevOps-Project-02/html-web-app)

## Pre-Deployment

Customize the application dependencies mentioned below on AWS EC2 instance and create the Golden AMI.

1. AWS CLI
2. Install Apache Web Server
3. Install Git
4. Cloudwatch Agent
5. Push custom memory metrics to Cloudwatch.
6. AWS SSM Agent

# AWS Scalable VPC Architecture Deployment Guide

This guide provides step-by-step instructions for deploying a scalable and modular AWS VPC architecture as outlined in the project requirements. The architecture includes two VPCs connected via Transit Gateway, with public and private subnets, NAT Gateways, Internet Gateways, and a highly available web application deployment.

## Architecture Overview

The architecture consists of:
- **VPC 1 (192.168.0.0/16)**: For Bastion Host deployment
  - Public Subnets
  - Internet Gateway
  - Route Tables
  - Security Groups
  - Bastion Host with Elastic IP
  
- **VPC 2 (172.32.0.0/16)**: For highly available web application
  - Public Subnets (Multiple AZs)
  - Private Subnets (Multiple AZs)
  - Internet Gateway
  - NAT Gateway
  - Route Tables
  - Auto Scaling Group
  - Network Load Balancer
  
- **Connectivity**:
  - Transit Gateway connecting both VPCs
  - VPC Flow Logs with CloudWatch integration
  - Route53 for DNS management

## Prerequisites

Before beginning the deployment, ensure you have:

1. An active AWS account with appropriate permissions
2. AWS CLI installed and configured
3. Knowledge of basic AWS networking concepts
4. Source code for the web application

## Step 1: Create Golden AMI

First, we'll create a Golden AMI with all required dependencies:

1. Launch an EC2 instance with Amazon Linux 2
2. Connect to the instance and install dependencies:

```bash
# Update the system
sudo yum update -y

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Apache Web Server
sudo yum install -y httpd
sudo systemctl enable httpd

# Install Git
sudo yum install -y git

# Install CloudWatch Agent
sudo yum install -y amazon-cloudwatch-agent

# Configure CloudWatch Agent for custom memory metrics
# Create a temporary file in your home directory
cat > /tmp/amazon-cloudwatch-agent.json << 'EOF'
{
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF

# Then use sudo to copy it to the destination
sudo cp /tmp/amazon-cloudwatch-agent.json /opt/aws/amazon-cloudwatch-agent/etc/

sudo systemctl restart amazon-cloudwatch-agent

# Start CloudWatch Agent
sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl start amazon-cloudwatch-agent

# Verify SSM Agent is installed (comes pre-installed on Amazon Linux 2)
sudo systemctl status amazon-ssm-agent
```

3. Create an AMI from this instance:
   - In the EC2 console, select the instance
   - Actions > Image and templates > Create image
   - Provide a name and description like "WebApp-Golden-AMI"
   - Create Image

## Step 2: VPC Deployment

### 2.1 Create VPC 1 (Bastion Host VPC)

```bash
# Create VPC 1
VPC1_ID=$(aws ec2 create-vpc \
  --cidr-block 192.168.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Bastion-VPC}]' \
  --query Vpc.VpcId --output text)

# Enable DNS hostnames and support
aws ec2 modify-vpc-attribute --vpc-id $VPC1_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC1_ID --enable-dns-support

# Create public subnet in AZ 1a
PUBLIC_SUBNET1_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC1_ID \
  --cidr-block 192.168.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Bastion-Public-Subnet}]' \
  --query Subnet.SubnetId --output text)

# Create Internet Gateway
IGW1_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Bastion-IGW}]' \
  --query InternetGateway.InternetGatewayId --output text)

# Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway --internet-gateway-id $IGW1_ID --vpc-id $VPC1_ID

# Create route table for public subnet
PUBLIC_RT1_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC1_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Bastion-Public-RT}]' \
  --query RouteTable.RouteTableId --output text)

# Add route to Internet Gateway
aws ec2 create-route --route-table-id $PUBLIC_RT1_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW1_ID

# Associate route table with public subnet
aws ec2 associate-route-table --route-table-id $PUBLIC_RT1_ID --subnet-id $PUBLIC_SUBNET1_ID
```

### 2.2 Create VPC 2 (Application VPC)

```bash
# Create VPC 2
VPC2_ID=$(aws ec2 create-vpc \
  --cidr-block 172.32.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=App-VPC}]' \
  --query Vpc.VpcId --output text)

# Enable DNS hostnames and support
aws ec2 modify-vpc-attribute --vpc-id $VPC2_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC2_ID --enable-dns-support

# Create public subnets in AZ 1a and 1b
PUBLIC_SUBNET2A_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC2_ID \
  --cidr-block 172.32.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=App-Public-Subnet-1a}]' \
  --query Subnet.SubnetId --output text)

PUBLIC_SUBNET2B_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC2_ID \
  --cidr-block 172.32.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=App-Public-Subnet-1b}]' \
  --query Subnet.SubnetId --output text)

# Create private subnets in AZ 1a and 1b
PRIVATE_SUBNET2A_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC2_ID \
  --cidr-block 172.32.3.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=App-Private-Subnet-1a}]' \
  --query Subnet.SubnetId --output text)

PRIVATE_SUBNET2B_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC2_ID \
  --cidr-block 172.32.4.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=App-Private-Subnet-1b}]' \
  --query Subnet.SubnetId --output text)

# Create Internet Gateway
IGW2_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=App-IGW}]' \
  --query InternetGateway.InternetGatewayId --output text)

# Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway --internet-gateway-id $IGW2_ID --vpc-id $VPC2_ID

# Create route table for public subnets
PUBLIC_RT2_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC2_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=App-Public-RT}]' \
  --query RouteTable.RouteTableId --output text)

# Add route to Internet Gateway
aws ec2 create-route --route-table-id $PUBLIC_RT2_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW2_ID

# Associate route table with public subnets
aws ec2 associate-route-table --route-table-id $PUBLIC_RT2_ID --subnet-id $PUBLIC_SUBNET2A_ID
aws ec2 associate-route-table --route-table-id $PUBLIC_RT2_ID --subnet-id $PUBLIC_SUBNET2B_ID

# Create NAT Gateway (requires Elastic IP)
EIP_ID=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET2A_ID \
  --allocation-id $EIP_ID \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=App-NAT-GW}]' \
  --query NatGateway.NatGatewayId --output text)

# Wait for NAT Gateway to be available
echo "Waiting for NAT Gateway to be available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID

# Create route table for private subnets
PRIVATE_RT2_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC2_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=App-Private-RT}]' \
  --query RouteTable.RouteTableId --output text)

# Add route to NAT Gateway
aws ec2 create-route --route-table-id $PRIVATE_RT2_ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID

# Associate route table with private subnets
aws ec2 associate-route-table --route-table-id $PRIVATE_RT2_ID --subnet-id $PRIVATE_SUBNET2A_ID
aws ec2 associate-route-table --route-table-id $PRIVATE_RT2_ID --subnet-id $PRIVATE_SUBNET2B_ID
```

## Step 3: Set Up Transit Gateway

```bash
# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=Multi-VPC-TGW}]' \
  --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable" \
  --query TransitGateway.TransitGatewayId \
  --output text)

# Wait until the Transit Gateway is available
while true; do
  STATE=$(aws ec2 describe-transit-gateways --transit-gateway-ids $TGW_ID --query 'TransitGateways[0].State' --output text)
  if [ "$STATE" == "available" ]; then
    echo "Transit Gateway is now available."
    break
  else
    echo "Transit Gateway state: $STATE. Waiting..."
    sleep 10
  fi
done

# Create Transit Gateway attachment for VPC 1
TGW_ATTACH1_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC1_ID \
  --subnet-ids $PUBLIC_SUBNET1_ID \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Bastion-VPC-Attachment}]' \
  --query TransitGatewayVpcAttachment.TransitGatewayAttachmentId --output text)

# Create Transit Gateway attachment for VPC 2
TGW_ATTACH2_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC2_ID \
  --subnet-ids $PRIVATE_SUBNET2A_ID $PRIVATE_SUBNET2B_ID \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=App-VPC-Attachment}]' \
  --query TransitGatewayVpcAttachment.TransitGatewayAttachmentId --output text)

# Wait for attachments to be available
echo "Waiting for Transit Gateway attachments to be available..."

# To Get the Transit Gateway attachment Id
aws ec2 describe-transit-gateway-attachments

# Wait until the Transit Gateway attachment is available
TGW_ATTACH1_ID="tgw-attach-061f15f8d72f9e2c9"  # Replace with your actual Transit Gateway Attachment ID
while true; do
  STATE=$(aws ec2 describe-transit-gateway-attachments \
    --transit-gateway-attachment-ids $TGW_ATTACH1_ID \
    --query 'TransitGatewayAttachments[0].State' \
    --output text)

  if [ "$STATE" == "available" ]; then
    echo "Transit Gateway Attachment is now available."
    break
  elif [ "$STATE" == "failed" ] || [ "$STATE" == "deleted" ]; then
    echo "Transit Gateway Attachment is in a failed or deleted state. Exiting."
    exit 1
  else
    echo "Transit Gateway Attachment state: $STATE. Waiting..."
    sleep 10
  fi
done


TGW_ATTACH2_ID="tgw-attach-0e70e70c6db7e372c"  # Replace with your actual Transit Gateway Attachment ID
while true; do
  STATE=$(aws ec2 describe-transit-gateway-attachments \
    --transit-gateway-attachment-ids $TGW_ATTACH2_ID \
    --query 'TransitGatewayAttachments[0].State' \
    --output text)

  if [ "$STATE" == "available" ]; then
    echo "Transit Gateway Attachment is now available."
    break
  elif [ "$STATE" == "failed" ] || [ "$STATE" == "deleted" ]; then
    echo "Transit Gateway Attachment is in a failed or deleted state. Exiting."
    exit 1
  else
    echo "Transit Gateway Attachment state: $STATE. Waiting..."
    sleep 10
  fi
done

# Get Transit Gateway Route Table ID
TGW_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
  --query "TransitGatewayRouteTables[0].TransitGatewayRouteTableId" --output text)

# Create routes in VPC route tables to direct traffic through Transit Gateway
# Route from VPC 1 to VPC 2
aws ec2 create-route \
  --route-table-id $PUBLIC_RT1_ID \
  --destination-cidr-block 172.32.0.0/16 \
  --transit-gateway-id $TGW_ID

# Route from VPC 2 to VPC 1
aws ec2 create-route \
  --route-table-id $PRIVATE_RT2_ID \
  --destination-cidr-block 192.168.0.0/16 \
  --transit-gateway-id $TGW_ID
```

## Step 4: Set Up VPC Flow Logs

```bash
# Create CloudWatch Log Group
aws logs create-log-group --log-group-name /aws/vpc/flowlogs

# Create Log Streams for each VPC
aws logs create-log-stream --log-group-name /aws/vpc/flowlogs --log-stream-name bastion-vpc-flowlogs
aws logs create-log-stream --log-group-name /aws/vpc/flowlogs --log-stream-name app-vpc-flowlogs

# Create IAM Role for VPC Flow Logs
cat > flowlogs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

cat > flowlogs-permission-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF

FLOW_LOGS_ROLE_NAME="VPCFlowLogsRole"
FLOW_LOGS_ROLE_ARN=$(aws iam create-role \
  --role-name $FLOW_LOGS_ROLE_NAME \
  --assume-role-policy-document file://flowlogs-trust-policy.json \
  --query "Role.Arn" --output text)

aws iam put-role-policy \
  --role-name $FLOW_LOGS_ROLE_NAME \
  --policy-name VPCFlowLogsPolicy \
  --policy-document file://flowlogs-permission-policy.json

# Enable Flow Logs for VPC 1
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC1_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination "arn:aws:logs:$(aws configure get region):$(aws sts get-caller-identity --query Account --output text):log-group:/aws/vpc/flowlogs" \
  --deliver-logs-permission-arn $FLOW_LOGS_ROLE_ARN

# Enable Flow Logs for VPC 2
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC2_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination "arn:aws:logs:$(aws configure get region):$(aws sts get-caller-identity --query Account --output text):log-group:/aws/vpc/flowlogs" \
  --deliver-logs-permission-arn $FLOW_LOGS_ROLE_ARN
```

## Step 5: Create Security Groups

```bash
# Create Security Group for Bastion Host
BASTION_SG_ID=$(aws ec2 create-security-group \
  --group-name "Bastion-SG" \
  --description "Security Group for Bastion Host" \
  --vpc-id $VPC1_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=Bastion-SG}]' \
  --query "GroupId" --output text)

# Allow SSH access from anywhere to Bastion Host
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Create Security Group for Web Servers
WEBSERVER_SG_ID=$(aws ec2 create-security-group \
  --group-name "WebServer-SG" \
  --description "Security Group for Web Servers" \
  --vpc-id $VPC2_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=WebServer-SG}]' \
  --query "GroupId" --output text)

# Allow HTTP access from anywhere to Web Servers
aws ec2 authorize-security-group-ingress \
  --group-id $WEBSERVER_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH access only from Bastion Host to Web Servers
BASTION_VPC_CIDR="192.168.0.0/16"    #Enter the CIDR block of the Bastion Host VPC
aws ec2 authorize-security-group-ingress \
  --group-id $WEBSERVER_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr "$BASTION_VPC_CIDR"
```

## Step 6: Deploy Bastion Host

```bash
# Create key pair for SSH access
aws ec2 create-key-pair \
  --key-name "bastion-key" \
  --query "KeyMaterial" \
  --output text > bastion-key.pem

chmod 400 bastion-key.pem

# Launch Bastion Host
aws ec2 run-instances \
  --image-id ami-08b5b3a93ed654d19 \
  --instance-type t2.micro \
  --key-name "bastion-key" \
  --security-group-ids "$BASTION_SG_ID" \
  --subnet-id "$PUBLIC_SUBNET1_ID" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Bastion-Host}]' \
  --query "Instances[0].InstanceId" \
  --output text


# Allocate and associate Elastic IP for Bastion Host
BASTION_EIP_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=Bastion-EIP}]' \
  --query "AllocationId" --output text)

aws ec2 associate-address \
  --instance-id $BASTION_INSTANCE_ID \
  --allocation-id $BASTION_EIP_ID

BASTION_PUBLIC_IP=$(aws ec2 describe-addresses \
  --allocation-ids $BASTION_EIP_ID \
  --query "Addresses[0].PublicIp" --output text)

echo "Bastion Host Public IP: $BASTION_PUBLIC_IP"
```

## Step 7: Create S3 Bucket for Configuration

```bash
# Generate a unique bucket name
S3_BUCKET_NAME="webapp-config-$(date +%s)"

# Create S3 bucket
aws s3api create-bucket \
  --bucket $S3_BUCKET_NAME \
  --region $(aws configure get region) \   # IF it doesn't work please replace with project region ex. us-east-1

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket $S3_BUCKET_NAME \
  --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'

# Block public access
aws s3api put-public-access-block \
  --bucket $S3_BUCKET_NAME \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Upload a sample configuration file
cat > webapp-config.json << 'EOF'
{
  "app_name": "My Web Application",
  "version": "1.0.0",
  "environment": "production",
  "logging_level": "info"
}
EOF

aws s3 cp webapp-config.json s3://$S3_BUCKET_NAME/webapp-config.json

echo "S3 Bucket created: $S3_BUCKET_NAME"
```

## Step 8: Create IAM Role for EC2 Instances

```bash
# Create IAM policy for S3 access
cat > webapp-s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::$S3_BUCKET_NAME",
        "arn:aws:s3:::$S3_BUCKET_NAME/*"
      ]
    }
  ]
}
EOF

# Create IAM Role for EC2 instances
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

WEBAPP_ROLE_NAME="WebAppRole"
aws iam create-role \
  --role-name $WEBAPP_ROLE_NAME \
  --assume-role-policy-document file://ec2-trust-policy.json

# Create and attach policy for S3 access
WEBAPP_POLICY_ARN=$(aws iam create-policy \
  --policy-name "WebAppS3Policy" \
  --policy-document file://webapp-s3-policy.json \
  --query "Policy.Arn" --output text)

aws iam attach-role-policy \
  --role-name $WEBAPP_ROLE_NAME \
  --policy-arn $WEBAPP_POLICY_ARN

# Attach AWS managed policy for SSM
aws iam attach-role-policy \
  --role-name $WEBAPP_ROLE_NAME \
  --policy-arn "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"

# Create instance profile and add role to it
aws iam create-instance-profile \
  --instance-profile-name $WEBAPP_ROLE_NAME

aws iam add-role-to-instance-profile \
  --instance-profile-name $WEBAPP_ROLE_NAME \
  --role-name $WEBAPP_ROLE_NAME

INSTANCE_PROFILE_ARN=$(aws iam get-instance-profile \
  --instance-profile-name $WEBAPP_ROLE_NAME \
  --query "InstanceProfile.Arn" --output text)

echo "IAM Instance Profile ARN: $INSTANCE_PROFILE_ARN"
```

## Step 9: Create Launch Configuration

```bash
# Create user data script
cat > userdata.sh << EOF
#!/bin/bash
yum update -y

# Start CloudWatch agent
systemctl start amazon-cloudwatch-agent
systemctl enable amazon-cloudwatch-agent

# Clone application code from repository
cd /var/www/html
git clone https://github.com/NotHarshhaa/DevOps-Projects.git
cp -r DevOps-Projects/DevOps-Project-02/html-web-app/* /var/www/html/

# Download config from S3
aws s3 cp s3://$S3_BUCKET_NAME/webapp-config.json /var/www/html/config.json

# Set permissions
chown -R apache:apache /var/www/html

# Start Apache
systemctl start httpd
systemctl enable httpd
EOF

# Create Launch Configuration
GOLDEN_AMI_ID="ami-xxxxxxxxxxxxxxxxx"  # Replace with your Golden AMI ID

#Create Launch Template of the Golden AMI
aws ec2 create-launch-template \
  --launch-template-name "WebApp-LT" \
  --version-description "Initial version" \
  --launch-template-data "{
    \"ImageId\": \"$GOLDEN_AMI_ID\",
    \"InstanceType\": \"t2.micro\",
    \"KeyName\": \"bastion-key\",
    \"SecurityGroupIds\": [\"$WEBSERVER_SG_ID\"],
    \"IamInstanceProfile\": { \"Name\": \"$WEBAPP_ROLE_NAME\" },
    \"UserData\": \"$(base64 -w0 userdata.sh)\"
  }"


aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "WebApp-ASG" \
  --launch-template "LaunchTemplateName=WebApp-LT,Version=1" \
  --min-size 1 \
  --max-size 3 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$PRIVATE_SUBNET2A_ID,$PRIVATE_SUBNET2B_ID"
```

## Step 10: Create Target Group and Network Load Balancer

```bash
# Create Target Group
TG_ARN=$(aws elbv2 create-target-group \
  --name "WebApp-TG" \
  --protocol TCP \
  --port 80 \
  --vpc-id $VPC2_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-port 80 \
  --query "TargetGroups[0].TargetGroupArn" --output text)

# Create Network Load Balancer
NLB_ARN=$(aws elbv2 create-load-balancer \
  --name "WebApp-NLB" \
  --type network \
  --subnets $PUBLIC_SUBNET2A_ID $PUBLIC_SUBNET2B_ID \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn $NLB_ARN \
  --protocol TCP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Get NLB DNS name
NLB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $NLB_ARN \
  --query "LoadBalancers[0].DNSName" --output text)

echo "NLB DNS Name: $NLB_DNS"
```

## Step 11: Create Auto Scaling Group

```bash
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "WebApp-ASG" \
  --launch-configuration-name "WebApp-LC" \
  --min-size 2 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$PRIVATE_SUBNET2A_ID,$PRIVATE_SUBNET2B_ID" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --tags "Key=Name,Value=WebApp-Instance,PropagateAtLaunch=true"

# Create scaling policies
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "WebApp-ASG" \
  --policy-name "WebApp-ScaleOut" \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://<(cat <<EOF
{
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "TargetValue": 70.0,
  "DisableScaleIn": false
}
EOF
)
```

## Step 12: Update Route53 DNS

```bash
# Get hosted zone ID (replace with your actual hosted zone ID)
HOSTED_ZONE_ID="Z1234567890ABC"
DOMAIN_NAME="example.com"

# Create CNAME record
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file://<(cat <<EOF
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "webapp.$DOMAIN_NAME",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "$NLB_DNS"
          }
        ]
      }
    }
  ]
}
EOF
)

echo "DNS record created: webapp.$DOMAIN_NAME pointing to $NLB_DNS"
```

## Validation

### Bastion Host Access

To access the bastion host:

```bash
ssh -i bastion-key.pem ec2-user@$BASTION_PUBLIC_IP
```

### Accessing Private Instances via Bastion Host

To access a private instance through the bastion host:

1. First, copy your private key to the bastion host:
```bash
scp -i bastion-key.pem bastion-key.pem ec2-user@$BASTION_PUBLIC_IP:~/
```

2. SSH to the bastion host:
```bash
ssh -i bastion-key.pem ec2-user@$BASTION_PUBLIC_IP
```

3. From the bastion host, SSH to a private instance:
```bash
# Replace PRIVATE_IP with the actual private IP of an instance
ssh -i ~/bastion-key.pem ec2-user@PRIVATE_IP
```

### Accessing Instances via AWS Session Manager

1. Open the AWS Management Console
2. Navigate to EC2 > Instances
3. Select the instance you want to connect to
4. Click on "Connect"
5. Choose "Session Manager" tab
6. Click "Connect"

### Testing the Web Application

1. Open a web browser
2. Navigate to `http://webapp.example.com` (replace with your actual domain)
3. Verify that the web page loads correctly

## Troubleshooting

### Common Issues and Solutions

1. **VPC Connectivity Issues**:
   - Check Transit Gateway route tables
   - Verify security group rules
   - Test connectivity using ping or telnet

2. **Auto Scaling Issues**:
   - Check launch configuration parameters
   - Verify instance health checks
   - Review CloudWatch metrics for scaling triggers

3. **Web Application Access Issues**:
   - Verify NLB target group health
   - Check Apache service status on instances
   - Review security group
