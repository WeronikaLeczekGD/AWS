# AWS

## [1. Write one or several bash scripts (using AWS CLI) which:](#ad-1)

### [a. Create VPC, Subnet, Elastic Container Registry (ECR), EC2 instance, public IP address for EC2 instance, Security Groups](#ad-a)

### [b. Push a java spring-petlinic container image you built in “Docker” module to ECR](#ad-b)

### [c. ECR should be private](#ad-c)

### [d. Install Docker on the VM](#ad-d)

### [e. Run container on the VM](#ad-e)

## [2. Verify that application is accessible in your browser](#ad-2)

## 3. Remove resources

## Ad 1

I wrote one script doing all tasks:

## Ad a

```
#!/bin/bash

# Create VPC with tags
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --amazon-provided-ipv6-cidr-block --query 'Vpc.VpcId' --output text --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]")

## Create subnet in VPC
SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]" --query 'Subnet.SubnetId' --output text)

#Create Elastic Container Registry (ECR)
aws ecr create-repository --repository-name spring-petclinic --image-tag-mutability IMMUTABLE

## Create EC2 instance
INSTANCE_ID=$(aws ec2 run-instances --image-id ami-00874d747dde814fa --instance-type t2.micro --subnet-id $SUBNET_ID --security-group-ids $SECURITY_GROUP_ID --associate-public-ip-address --key-name spring-project-EC2 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]" --query 'Instances[0].InstanceId' --output text)

##Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]")

##Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

##Associate Public Subnet with route table
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 associate-route-table --subnet-id $SUBNET_ID --route-table-id $ROUTE_TABLE_ID

##Create Route to Internet Gateway
aws ec2 create-route --route-table-id $ROUTE_TABLE_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

## Alloccate Elastic IP for NAT Gateway
EIP_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

## Create NAT Gateway
aws ec2 create-nat-gateway --subnet-id $SUBNET_ID --allocation-id $EIP_ID --output text --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]"

##public IP address for EC2 instance, which will be used to connect to the instance via SSH
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[*].Instances[*].PublicIpAddress' --output text)

## Create security group
SECURITY_GROUP_ID=$(aws ec2 create-security-group --group-name "ec2" --description "Security group for EC2 instance" --vpc-id $VPC_ID --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=spring EC2},{Key=Owner,Value=wleczek},{Key=Project,Value=2022_intership_wro}]" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 
```

## Ad b

Pushing a java spring-petlinic container image you built in “Docker” module to ECR

```
#login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 113304117666.dkr.ecr.us-east-1.amazonaws.com

#tag image
docker tag wleczek/mr:32 113304117666.dkr.ecr.us-east-1.amazonaws.com/spring-petclinic:latest

#push image to ECR
docker push 113304117666.dkr.ecr.us-east-1.amazonaws.com/spring-petclinic:latest
```
## Ad c

My private ECR:

![Screenshot 2023-02-06 at 11 57 59](https://user-images.githubusercontent.com/114099418/216954583-9b68a2fe-61c6-4c36-9542-c64bb780ba25.png)

## Ad d

Installing Docker on the VM:

```
#attach IAM role to EC2 instance
aws ec2 associate-iam-instance-profile --instance-id $INSTANCE_ID --iam-instance-profile Name=EcrRegistryFullAccessEc2

## Install Docker on EC2 instance
ssh -i "/Users/wleczek/Downloads/spring-project-EC2.pem" ubuntu@$PUBLIC_IP 'sudo apt-get update && sudo apt-get install docker.io -y'
```

## Ad e

Running container on EC2 instance:

```
## Run container on EC2 instance
ssh -i "/Users/wleczek/Downloads/spring-project-EC2.pem" ubuntu@$PUBLIC_IP 'docker run -p 8080:8080 113304117666.dkr.ecr.us-east-1.amazonaws.com/spring-petclinic:latest'

```

## Ad 2

Verifying that application is accesible in my browser:

![Screenshot 2023-02-06 at 12 05 43](https://user-images.githubusercontent.com/114099418/216955994-f0298a3b-0ed8-4142-a1be-c375f4b6df96.png)
