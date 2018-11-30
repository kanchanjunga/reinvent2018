# AWS PrivateLink:

- [AWS PrivateLink](https://aws.amazon.com/privatelink/) enables service consumer to privately connect their VPC to supported AWS services, AWS Marketplace partner services and services hosted by other AWS customers and partners in their own VPCs without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection.

- You can create your own application in your VPC and configure it as an AWS PrivateLink-powered services. Other AWS principals can create a connection from their VPC to your VPC endpoint service using interface VPC endpoint.

## Objective:

- In this we will focus on creating your own [VPC Endpoint Service](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html) in your own VPC and allowing other business units (AWS principals) to create [Interface VPC Endpoint](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html) connection from their VPC to VPC Endpoint Service.
  - We will call VPC hosting the service as the service provider VPC and VPC consuming the service as service consumer VPC

- Service provider VPC and service consumer VPC will have overlapping IP address.

![](images/aws_privatelink.png)

### Host a service:

#### Host 'Hello World' webserver

1. Create VPC
2. Create three public subnets, one in each availability zone
   - Host a service in a private subnet if service should not be exposed to internet. We are creating public subnet only for the purose of demonstration.
3. Create custom route table and associate three subnets created in step (2) to this newly created custom route table
4. Create custom security group to allow port 22 and port 80 traffic
5. Create EC2 instance in each availability zone
   - Configure webserver to render simple "Hello World" page
   - One of the instance will have an **overlapping ip address**
6. Create Network Load Balancer to frontend EC2 instance created in step (5)
7. **Create VPC Endpoint Service**

### Access the service:

1. Create VPC
2. Create one public subnet in one of the availability zone
   - Create a private subnet if resource should not be exposed to internet. We are creating public subnet only for the purose of demonstration.
3. Create custom route table and associate the subnet created in step (2) to this newly created custom route table
4. Create custom security group to allow port 22 and port 80 traffic
5. Create EC2 instance with an **overlapping ip address**
6. **Create Interface VPC Endpoint**
7. Access the service

### Understand concepts:

- AWS PrivateLink
- Overlapping IP adddress
- How AWS PrivateLink allows to connect two VPCs with overlapping ip address

## Workshop setup:

### Create ssh key-pair:

- **Skip this step** if you want to use your existing ssh key-pair

#### Using Amazon EC2 console

- To create ssh key-pair using Amazon EC2 Console, click on [Amazon EC2 console](https://eu-west-1.console.aws.amazon.com/ec2/).
  - Documentation can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair)
  - Verify region is set to eu-west-1 (Ireland)

### Configure AWS CLI:

- For this session will use AWS CLI to configure required AWS resources

#### Install:

If you don't have AWS CLI installed or don't have up to date version, follow installion/upgrade instruction found [here]().

#### Configure:

- Verify installation:
```
aws version
```

- Configure AWS CLI:
```
aws configure
```
```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE --> This should be your IAM User access key ID. IAM User that you will use to configure AWS resources in this document
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY --> This should be your IAM User access key ID. IAM User that you will use to configure AWS resources in this document
Default region name [None]: eu-west-1 --> Region of your choice, for this session we will use eu-west-1 (Ireland) region
Default output format [None]: json --> format of your choice
```

### Create underlying infrastructure using AWS CloudFormation:

- Launch [Service Provider AWS CloudFormation Template in eu-west-1 (Ireland) region](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?templateURL=https://s3-eu-west-1.amazonaws.com/net316-builder-session-eu-west-1/net316ServiceProvider.json) to create:
  - VPC
  - subnets
  - route table
  - security group
  - EC2 instances with webserver
- Launch [Service Consumer AWS CloudFormation Template in eu-west-1 (Ireland) region](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?templateURL=https://s3-eu-west-1.amazonaws.com/net316-builder-session-eu-west-1/net316ServiceConsumer.json) to create:
  - VPC
  - subnet
  - route table
  - security group
  - EC2 instance

#### AWS CloudFormation Templates used to create above resources can be found here:

- [Service Provider AWS CloudFormation Template](https://s3-eu-west-1.amazonaws.com/net316-builder-session-eu-west-1/net316ServiceProvider.json)
- [Service Consumer AWS CloudFormation Template](https://s3-eu-west-1.amazonaws.com/net316-builder-session-eu-west-1/net316ServiceConsumer.json)

### Create VPC Endpoint Service (hosting your service)

1. Create AWS Network Load Balancer:
```
aws elbv2 create-load-balancer \
--name net316-vpca-nlb1 \
--type network --ip-address-type ipv4 \
--scheme internal \
--subnets <subnet-id1 subnet-id2 subnet-id3> \
--region eu-west-1
```

2. Create target group:
```
aws elbv2 create-target-group \
--name webServerGroup1 \
--protocol TCP \
--port 80 \
--vpc-id <vpc-id> \
--region eu-west-1
```

3. Register targets (ec2 instances) with target group:
```
aws elbv2 register-targets \
--target-group-arn <target group arn from step(2)> \
--targets Id=<Webserver1-id> Id=<Webserver2-id> Id=<Webserver3-id> \
--region eu-west-1
```

4. Create listener:
```
aws elbv2 create-listener \
--load-balancer-arn <nlb-arn from step(1)> \
--protocol TCP \
--port 80 \
--default-actions Type=forward,TargetGroupArn=<target-group-arn from step(2)> \
--region eu-west-1
```

5. Create VPC Endpoint Service:
```
aws ec2 create-vpc-endpoint-service-configuration \
--network-load-balancer-arn <nlb-arn from step(1)> \
--no-acceptance-required \
--region eu-west-1
```

6. Describe/view VPC Endpoint Service configuration:
```
aws ec2 describe-vpc-endpoint-services \
--service-name  <Service ARN from step(50> \
--region eu-west-1
```

### Create Interface VPC Endpoint Service (access hosted service):

1. Create Interface VPC Endpoint
   - Create the endpoing in the same subnet as the EC2 instance:
```
aws ec2 create-vpc-endpoint \
--vpc-endpoint-type Interface \
--service-name <Service ARN from step(5)> \
--subnet-id <subnet-id1> \
--vpc-id <net316 service consumer vpc-id> \
--security-group-ids <net316-service-consumer-security-group-id> \
--region eu-west-1
```

2. Dscribe/view VPC Endpoint configuration:
```
awsws ec2 describe-vpc-endpoints \
--vpc-endpoint-ids <vpc-endpoint-id created above> --region eu-west-1
```

3. Access the service:
   - ssh in to EC2 instance and issue a curl command:
```
curl <vpc-endpoint-dns-name from step(1)>
```

### Clean up (delete AWS resources):

#### In Service Consumer environment:

1. Delete VPC Endpoint
```
awsws ec2 delete-vpc-endpoints \
--vpc-endpoint-ids <vpc-endpoint-id> \
--region eu-west-1
```

2. Delete [Service Consumer AWS CloudFormation Template](https://eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks)

#### In Service Provider environment:

1. Delete VPC Endpoint Service:
```
aws ec2 delete-vpc-endpoint-service-configurations \
--service-ids <vpce-endpoint-service-id> \
--region eu-west-1
```

2. Delete Listener

```
aws elbv2 delete-listener \
--listener-arn <listener-arn> \
--region eu-west-1
```

3. Delete Target Group
```
aws elbv2 delete-target-group \
--target-group-arn <target-group-arn> \
--region eu-west-1
```

4. Delete AWS Network Load Balancer
```
aws elbv2 delete-load-balancer \
--load-balancer-arn <nlb-load-balancer-ar> \
--region eu-west-1
```

5. Delete [Service Provider AWS CloudFormation Template](https://eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks)

## Considerations:

- Works with overlapping ip address
- Only service consumer (client) can intiate the connection. Strict Client to Server Access
- A single service you want to share to many VPCs, but may lack control or trust of the consumers
- Scales to thousands of consumers
- Simple to setup. Do not require NAT, IGW, VPN or DX
- Traffic stays private, does not traverse internet
- As of now supports only TCP

