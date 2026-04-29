## AWS NAT Instance Lab: Accessing Private Subnets
This repository contains a CloudFormation template to deploy a VPC architecture with a custom NAT Instance. This setup allows an EC2 instance in a private subnet to access the internet for updates while remaining inaccessible from the public web.
##  Architecture Overview

* VPC: 10.0.0.0/16
* Public Subnet: Hosts the NAT Instance and a Public Bastion Server.
* Private Subnet: Hosts a Private Server with no direct internet access.
* NAT Instance: An Ubuntu 24.04 instance configured with iptables to perform IP Masquerading.

##  Deployment

   1. Prerequisites:
   * An AWS account.
      * An EC2 Key Pair named NAT created in the ap-south-1 (Mumbai) region.
   2. CloudFormation:
   * Upload the template.yaml to the CloudFormation console.
      * Deploy the stack.
      * Note the PublicServerIP and PrivateServerPrivateIP from the Outputs tab.
   
------------------------------
##  Accessing the Private Server
Since the private server has no public IP, you must "jump" through the Public Server.
## Method 1: SSH Agent Forwarding (Recommended)
This method allows you to use your local private key to connect from the bastion to the private server without ever uploading the key to the cloud.

   1. Add your key to the agent:
   
   ssh-add -K NAT.pem
   
   2. Connect to the Public Server with forwarding enabled:
   
   ssh -A ubuntu@<PublicServerIP>
   
   3. From the Public Server, jump to the Private Server:
   
   ssh ubuntu@<PrivateServerPrivateIP>
   
   
## Method 2: Manual Key Copy (Not for Production)

   1. Copy the key to the Public Server:
   
   scp -i NAT.pem NAT.pem ubuntu@<PublicServerIP>:/home/ubuntu/
   
   2. SSH into Public Server:
   
   ssh -i NAT.pem ubuntu@<PublicServerIP>
   
   3. SSH into Private Server:
   
   chmod 400 NAT.pem
   ssh -i NAT.pem ubuntu@<PrivateServerPrivateIP>
   
   
------------------------------
##  Verifying Internet via NAT
Once inside the Private Server, run the following command to verify that it can reach the internet through the NAT instance:

ping -c 4 google.com

If you see replies, the NAT Instance is successfully routing traffic from your private subnet!
------------------------------
##  Troubleshooting

* Rollback Error: Ensure your Key Pair is named exactly NAT and you are in the ap-south-1 region.
* Connection Timeout: Check that SourceDestCheck is set to false on the NAT Instance in the AWS Console.
* No Internet in Private Subnet: Ensure the PrivateRouteTable has a route 0.0.0.0/0 pointing to the NAT Instance ID.

------------------------------
## CloudFormation yaml Code for NAT instance:

## 📄 CloudFormation Template

Copy the code below and save it as `nat-lab.yaml`.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC with Public/Private Subnets, NAT Instance, and Hardcoded NAT Key

Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Default: NAT
    Description: Name of the existing KeyPair (Defaulted to NAT)

Resources:
  # 1. VPC & IGW
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - {Key: Name, Value: NAT-Lab-VPC}

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - {Key: Name, Value: NAT-Lab-IGW}

  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # 2. Subnets
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - {Key: Name, Value: Public-Subnet}

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      Tags:
        - {Key: Name, Value: Private-Subnet}

  # 3. Routing
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PrivateAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet
      RouteTableId: !Ref PrivateRouteTable

  # 4. Security Groups
  MainSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and internal VPC traffic
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: -1 
          CidrIp: 10.0.0.0/16

  # 5. NAT Instance (Ubuntu 24.04 in ap-south-1)
  NATInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0ec10929233384c7f
      SubnetId: !Ref PublicSubnet
      KeyName: !Ref KeyName
      SourceDestCheck: false 
      SecurityGroupIds: [!Ref MainSecurityGroup]
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          export DEBIAN_FRONTEND=noninteractive
          apt-get update -y
          apt-get install -y iptables-persistent
          sysctl -w net.ipv4.ip_forward=1
          echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
          IFACE=$(ip route | grep default | awk '{print $5}')
          iptables -t nat -A POSTROUTING -o $IFACE -j MASQUERADE
          netfilter-persistent save
      Tags:
        - {Key: Name, Value: NAT-Instance}

  # 6. Private Route (Targets NAT Instance)
  PrivateRoute:
    Type: AWS::EC2::Route
    DependsOn: NATInstance
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      InstanceId: !Ref NATInstance

  # 7. Application Instances
  PublicServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0ec10929233384c7f
      SubnetId: !Ref PublicSubnet
      KeyName: !Ref KeyName
      SecurityGroupIds: [!Ref MainSecurityGroup]
      Tags:
        - {Key: Name, Value: Public-Server}

  PrivateServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0ec10929233384c7f
      SubnetId: !Ref PrivateSubnet
      KeyName: !Ref KeyName
      SecurityGroupIds: [!Ref MainSecurityGroup]
      Tags:
        - {Key: Name, Value: Private-Server}

Outputs:
  PublicServerIP:
    Description: Use this to SSH into the public network
    Value: !GetAtt PublicServer.PublicIp
  PrivateServerPrivateIP:
    Description: Internal IP of the private server
    Value: !GetAtt PrivateServer.PrivateIp
  NATInstancePublicIP:
    Value: !GetAtt NATInstance.PublicIp
```
## Screenshots
keypair
<img width="1920" height="1080" alt="keypair" src="https://github.com/user-attachments/assets/55450a53-1481-47da-8dec-fb9872bf1ef2" />

VPC Created through CloudFormation

<img width="1920" height="1080" alt="Screenshot (529)" src="https://github.com/user-attachments/assets/615b4edf-519e-4816-a1d3-b95bc5f40057" />

<img width="1920" height="1080" alt="Screenshot (530)" src="https://github.com/user-attachments/assets/2691c456-6a10-4e18-9f53-66f159a86db2" />

subnets

<img width="1920" height="1080" alt="Screenshot (531)" src="https://github.com/user-attachments/assets/539f380d-e204-46fd-835a-dc75f00efc27" />

Internet Gateway

<img width="1920" height="1080" alt="Screenshot (532)" src="https://github.com/user-attachments/assets/93bcf799-7239-428e-8b4f-3fe0eadf5d5b" />

Instances Created

<img width="1920" height="1080" alt="Screenshot (500)" src="https://github.com/user-attachments/assets/b5a9f942-3c3a-4b2c-9ecb-d5a225b335fc" />

Private Instance in Private Subnet Created

<img width="1920" height="1080" alt="Screenshot (501)" src="https://github.com/user-attachments/assets/7eacbe8f-78c4-495e-bae6-1eacb306150a" />

NAT Instance Created

<img width="1920" height="1080" alt="Screenshot (502)" src="https://github.com/user-attachments/assets/88c7644d-6f24-45ba-9d76-72fdcd991e33" />

PEM key transfer to NAT instance

<img width="1920" height="1080" alt="Screenshot (504)" src="https://github.com/user-attachments/assets/2c62c25a-fc23-410f-95a2-8401ea1acd33" />

Accessing Private Instance in Public Subnet through NAT Instance

<img width="1920" height="1080" alt="Screenshot (526)" src="https://github.com/user-attachments/assets/7274aed3-d660-4db1-918c-d96e2d820082" />

Private Instance downloadinf packages

<img width="1920" height="1080" alt="Screenshot (527)" src="https://github.com/user-attachments/assets/842e9320-36a4-4677-97aa-1257309d649a" />

## Note: 
  As the private instance is downloading packages, it is clear that one way internet is being accessed through NAT Instance.

