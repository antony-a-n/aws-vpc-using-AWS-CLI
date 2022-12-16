# Creating VPC via AWS-CLI
==================================

Hello folks,

  Infra management is a big task for every company. from choosing the right requirements to selecting the right tools to create and manage it.
 In most companies, infra management is done via terraform. Now we are trying to configure a basic VPC with AWS CLI for our testing purposes and to compare both of the tools.
 
 ## Resources created by this code
-   2 public subnets
-	1 private subnet
-	1 public route table
-	1 private route table
-	1 NAT gateway
-	1 Internet gateway
-	1 elastic IP
-	1 Bastion instance
-	1 front-end instance
-	1 backend -instance
	
#### Before proceeding please make sure that you have configured an IAM user with programmatic access EC2 full permission.

	
FYI, We will keep our bastion and front-end server in public subnets, and the backend server will be placed in the public subnet and it will connect to the internet via the NAT gateway.

This setup can be implemented to  host wordpress website by keeping the as database server in the private subnet and webserver in  the publich subnet and both can be accessed from the bsation server.

So let's start the process by configuring the AWS CLI  on the server, you should have the **Access key ID** and **Secret access key** of the IAM user to configure this.
# 1. Configuring AWSCLI	
===============================

You can use the  **aws configure** command to configure AWS CLI on your server., provide the required details.
	
```
[root@testserver ~]# aws configure 
AWS Access Key ID [None]: XXXXXXXXXXXXXXX 
AWS Secret Access Key [None]: XXXXXXXXXXXXXX 
Default region name [None]: ap-south-1 
Default output format [None]: json 
[root@testserver ~]#	
```
	
Once we successfully configured awscli, now we can proceed with creating VPC
## 2.VPC creation
===================

For Creating VPC, you can use the following command, In this example, we are configuring a VPC with CIDR block "10.0.0.0/16"
```
[root@ip-172-31-47-248 ~]# aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text 
```

You will get the output as follows

```
[root@ip-172-31-47-248 ~]# aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text
vpc-024b7015b71d95647
[root@ip-172-31-47-248 ~]#
```
You have created a VPC with ID "vpc-024b7015b71d95647 "

It's better to add tags to the resources created for easy identification in the future. **create-tags**  option can be used for that. In our example, we are using the  Key as Name and Value as TestVPC. **--resources** is used to specify our VPCID.

```
[root@testserver ~]# aws ec2 create-tags --resources vpc-024b7015b71d95647 --tags Key=Name,Value=TestVPC
```
# 3.Creating subnets
==========================

So once you have created the VPC and added the required tags, you can now proceed with making the subnets, We need  2 public subnets and 1 private subnet. So we are creating 3 subnets.

You have to be careful with choosing the CIDR range for the subnets. You can use **create-subnet**  to create the subnets. 

```
[root@testserver ~]# aws ec2 create-subnet --vpc-id vpc-024b7015b71d95647 --cidr-block 10.0.0.0/18 --query Subnet.SubnetId --output json 
"subnet-018afc6fad2baca84" 
[root@testserver ~]# aws ec2 create-subnet --vpc-id vpc-024b7015b71d95647 --cidr-block 10.0.64.0/18 --query Subnet.SubnetId --output json 
"subnet-085df01e6b2dbeb7c" 
[root@testserver ~]# aws ec2 create-subnet --vpc-id vpc-024b7015b71d95647 --cidr-block 10.0.128.0/18 --query Subnet.SubnetId --output json 
"subnet-082e68f430592fa3b" 
[root@testserver ~]#
```

Add tags to the created subnets. replace the subnetID with yours

```
[root@testserver ~]# aws ec2 create-tags --resources subnet-018afc6fad2baca84 --tags Key=Name,Value=subnet1 
[root@testserver ~]# aws ec2 create-tags --resources subnet-085df01e6b2dbeb7c --tags Key=Name,Value=subnet2 
[root@testserver ~]# aws ec2 create-tags --resources subnet-082e68f430592fa3b --tags Key=Name,Value=subnet3 
```
# 4.Creating internet gateway
======================================

We are routing the internet traffic in and out to our VPC via the internet gateway. So in the next step, we will create the internet gateway. 
**create-internet-gateway** can be used for creating IGW.

```
[root@testserver ~]# aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text 
igw-01f91a2838341f868 
[root@testserver ~]#
```

You will get an output like the above result,

Once you successfully created the internet gateway, we need to attach the internet gateway to our VPC.

**attach-internet-gateway** is used to  attach IGW to VPC

Please note that you need to replace the VPCid and IGWid with your corresponding values.

```
[root@testserver ~]# aws ec2 attach-internet-gateway --vpc-id vpc-024b7015b71d95647 --internet-gateway-id igw-01f91a2838341f868
```

Success, now you can proceed with creating  route tables,

# 5.Creating route tables
=========================

We are placing one of our subnets as private, and the other two subnets will be public, so we have to create separate route tables for the subnets.

**create-route-table** can be used for this.

```
[root@testserver ~]# aws ec2 create-route-table --vpc-id vpc-024b7015b71d95647 --query RouteTable.RouteTableId --output text 
rtb-0d1d514a5b11d5047 
[root@testserver ~]# aws ec2 create-route-table --vpc-id vpc-024b7015b71d95647 --query RouteTable.RouteTableId --output text 
rtb-07c636522e8758a91 
[root@testserver ~]#
```

Now we need to route the traffic through  the route table via the IGW  for the public subnets.

```
[root@testserver ~]# aws ec2 create-route --route-table-id rtb-0d1d514a5b11d5047 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-01f91a2838341f868
```

If the command executes successfully, you will get a message as follows
```
{ 
    "Return": true 
}
```

In the next step, we are attaching our public subnets to this route table. In this example, we are attaching 10.0.64.0/18 and 10.0.0.0/18  to this route table. 

Don't forget to replace the subnet ID with the corresponding value, **associate-route-table** command is used to attach the route table to the subnet.

```
[root@testserver ~]# aws ec2 associate-route-table  --subnet-id subnet-018afc6fad2baca84 --route-table-id rtb-0d1d514a5b11d5047
[root@testserver ~]# aws ec2 associate-route-table  --subnet-id subnet-085df01e6b2dbeb7c --route-table-id rtb-0d1d514a5b11d5047
```

You will get a message which indicates the subnets were attached successfully like the following.

```
{ 
    "AssociationState": { 
        "State": "associated" 
    }, 
    "AssociationId": "rtbassoc-0bc8c621ef9ba7bac" 
}
```

We are keeping 10.0.128.0/18 as a private subnet, so we are routing the traffic via a NAT gateway and we are placing the NAT gateway in the public subnet.
# 6.Creating NAT gateway
================================

 For configuring the NAT gateway, you need an elastic IP. You can get it using the following command **allocate-address**

```
aws ec2 allocate-address --domain vpc
```

-- domain defines the playground to your elastic IP address, If you are purchasing it for ec2-classic, then you need to add the value as standard/
We are using it within the VPC, so we can choose the value as PVC.

```
[root@testserver ~]# aws ec2 allocate-address --domain vpc
```

If it is allocated without any issues, you will be getting the following output

```
{ 
    "Domain": "vpc", 
    "PublicIpv4Pool": "amazon", 
    "PublicIp": "3.108.245.5", 
    "AllocationId": "eipalloc-0b83025c31c49a5e0", 
    "NetworkBorderGroup": "ap-south-1" 
}
```

Now it's time to attach the elastic IP to our NAT gateway and place it in the public subnet, we are keeping the NAT gateway in the 10.0.64.0/18 subnet.

Replace the subnet ID and allocation ID with corresponding values

```
[root@testserver ~]# aws ec2 create-nat-gateway --subnet-id subnet-082e68f430592fa3b --allocation-id eipalloc-0b83025c31c49a5e0
```

The outcome will be as follows,
```
{ 
    "NatGateway": { 
        "NatGatewayAddresses": [ 
            { 
                "AllocationId": "eipalloc-0b83025c31c49a5e0" 
            } 
        ], 
        "VpcId": "vpc-024b7015b71d95647", 
        "State": "pending", 
        "NatGatewayId": "nat-08709892873009668", 
        "SubnetId": "subnet-082e68f430592fa3b", 
        "CreateTime": "2022-12-15T20:06:36.000Z" 
    }, 
    "ClientToken": "6a973d33-ee51-41c8-bcd1-c684bfedb3a1" 
}
```

Next, we need to attach our private subnet to the route table,

```
[root@testserver ~]# aws ec2 associate-route-table --subnet-id subnet-082e68f430592fa3b --route-table-id rtb-07c636522e8758a91
```

Replace the subnet ID with the subnet ID of your private subnet.

You will get a message as follows once it is associated successfully.

```
{ 
    "AssociationState": { 
        "State": "associated" 
    }, 
    "AssociationId": "rtbassoc-0c9d0719f743ec877" 
}
```

Now we can create a route entry to make the traffic to the internet from this subnet via the NAT gateway.

```
[root@testserver ~]# aws ec2 create-route --route-table-id rtb-07c636522e8758a91 --destination-cidr-block 0.0.0.0/0 --gateway-id nat-08709892873009668
```

Replace the nat gateway id with the corresponding value.

You will get a success message like the following.

```
{ 
    "Return": true 
}
```

Since we are keeping the subnets 10.0.0.0/18 and 10.0.64.0/18 as public subnets when an instance is launched inside these subnets they need to be associated with a public IP address.

for that, we can use the **modify-subnet-attribute**

don't forget to replace the subnet ID.

```
[root@testserver ~]# aws ec2 modify-subnet-attribute --subnet-id subnet-018afc6fad2baca84 --map-public-ip-on-launch 
[root@testserver ~]# aws ec2 modify-subnet-attribute --subnet-id subnet-085df01e6b2dbeb7c --map-public-ip-on-launch

```
# 7.Creating security groups & adding rules to the groups

Now we have to create the required security group which acts like a firewall. In our VPC setup, we are launching 3 instances, a bastion server, a fronted server which is going to place in the public subnets, and a DB server placed in the private subnet. For the bastion server, only port 22 is open and for the front-end server port, 80 will accept the traffic from anywhere, and 22 is from the bastion server, and for the DB server port 22 is accessible from the bastion server, and 3306 from the frontend server.

To satisfy the above requirement, We are creating 3 security groups for these instances.

**create-security-group** is used to create security groups.

```
root@testserver ~]# aws ec2 create-security-group --group-name bastion --description "security group with port 22" --vpc-id vpc-024b7015b71d95647
{
    "GroupId": "sg-0bc8a174fba94e2e9"
}
[root@testserver ~]# aws ec2 create-security-group --group-name webserver --description "security group with port 22 and 80" --vpc-id vpc-024b7015b71d95647
{
    "GroupId": "sg-0eb8bfd79d3513ed3"
}
[root@testserver ~]# aws ec2 create-security-group --group-name dbserver --description "security group with port 22 and 3306" --vpc-id vpc-024b7015b71d95647
{
    "GroupId": "sg-0b6ae72007790dc70"
}
[root@testserver ~]#
```

Now you need to add inbound rules to the security groups.

For the bastion server, port 22 is open and it can be accessed from any IP, which is very vulnerable to attack. In the production environment, you need to add put your Ip with a --cidr range.

To add rules to the security groups authorize-security-group-ingress can be used.

```
[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0bc8a174fba94e2e9 --protocol tcp --port 22 --cidr 0.0.0.0/0
```

For the front-end server, we are opening ports 80,443 and 22. Port 80 and 443 can be accessed from anywhere and 22 is accessible from the bastion server.

```

[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0eb8bfd79d3513ed3 --protocol tcp --port 80 --cidr 0.0.0.0/0
[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0eb8bfd79d3513ed3 --protocol tcp --port 443 --cidr 0.0.0.0/0
```

For enabling ssh from the bastion server, we are using the security group as the source. --source-group can be used for that.

```
[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0eb8bfd79d3513ed3 --protocol tcp --port 22 --source-group sg-0bc8a174fba94e2e9
```

For the database server, we need port 22,3306 needs to be enabled.

```
[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0b6ae72007790dc70 --protocol tcp --port 22 --source-group sg-0bc8a174fba94e2e9
```

for 3306, we need access to this port only from the web server, so specify the source group with the security group ID of the webserver.

```
[root@testserver ~]# aws ec2 authorize-security-group-ingress --group-id sg-0b6ae72007790dc70 --protocol tcp --port 3306 --source-group sg-0eb8bfd79d3513ed3
```
# 8.Creating keypair

Now we need to create a key for connecting to our servers.
```
[root@testserver ~]# aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > newkey.pem
```

You have to change the permission  of the key to 400

```
[root@testserver ~]#chmod 400 newkey.pem
```

Before proceeding, we need enableDnsHostnames attribute for enabling the DNS hostnames for instances launched in the VPC.

```
[root@testserver ~]#aws ec2 modify-vpc-attribute --vpc-id vpc-024b7015b71d95647 --enable-dns-support "{\"Value\":true}"
[root@testserver ~]#aws ec2 modify-vpc-attribute --vpc-id vpc-024b7015b71d95647 --enable-dns-hostnames "{\"Value\":true}"
```
# 9. Launching instances in the new  VPC
 
Now we can launch our instances in our VPC.

We are launching the bastion server in the subnet "10.0.64.0/18" and the front-end server in the "10.0.0.0/18" subnet.

You can choose the AMI as per your requirement. In this  example, we are launching our instances with Amazon Linux AMI with ami-id  "ami-074dc0a6f6c764218"

we are choosing the type as t2.micro and instance count to 1.

```
[root@testserver ~]# aws ec2 run-instances --image-id ami-074dc0a6f6c764218 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-0bc8a174fba94e2e9 --subnet-id subnet-085df01e6b2dbeb7c
```
Once you successfully executed the command, it will display the full details of the instance.

Now it's the turn for the front-end instance.

we only need to update the security group ID and subnet ID.

```
[root@testserver ~]# aws ec2 run-instances --image-id ami-074dc0a6f6c764218 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-0eb8bfd79d3513ed3 --subnet-id subnet-018afc6fad2baca84
```

Finally, we are creating the DB server in the private subnet "10.0.128.0/18" with the associated security group.

```
[root@testserver ~]# aws ec2 run-instances --image-id ami-074dc0a6f6c764218 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-0b6ae72007790dc70 --subnet-id subnet-082e68f430592fa3b
```

Now  we can add Tags for the created instances.

```
[root@testserver ~]#aws ec2 create-tags --resources i-098c6a65873a99278 --tags Key=Name,Value=bastion 
[root@testserver ~]#aws ec2 create-tags --resources i-0dc7f42d36a763413 --tags Key=Name,Value=dbserver 
[root@testserver ~]#aws ec2 create-tags --resources i-0f569f94tr54d43fa --tags Key=Name,Value=webserver
```


Once you created the instances  you will be able to access the servers with the Key you have generated. 
