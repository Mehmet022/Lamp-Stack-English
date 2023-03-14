By creating a VPC via AWS, install a LAMP stack on Linux machines.

Let's start by creating a VPC with the appropriate /CIDR for the project. After creating the VPC, select it and go to Actions => Edit VPC Setting => Enable DNS hostnames.

To enable our network to access the internet, we need to create an Internet Gateway and attach it to the VPC.

Next, let's create the subnets. Create two public subnets and two private subnets in two AZs according to the /CIDR specified in the project. To enable automatic IP allocation for the public subnets, select them one by one and go to Actions => Edit Subnet Settings => Enable Auto Assign IP.

Filter the created VPC and set the default Route table to Public. Then, create a new Route Table and name it Private. Associate the created subnets with the public and private routes.

To allow machines in the public subnets to access the internet, select the Internet Gateway as the Target in Routes => Edit Routes => Add Routes => Destinations => 0.0.0.0/0.

Note: Use a NAT instance or NAT Gateway for private machines to access the internet.

Let's create the security groups:

First, let's create a security group for the NAT instance. Select the created VPC and enter the inbound rules.

SSH ===============> Source: My IP
HTTP ==============> Anywhere-IPv4
HTTPS =============> Anywhere-IPv4
ICMP IPv4 =============> Anywhere-IPv4 (opened to determine whether we are connected to the internet by pinging. It can be removed optionally.)
Outbound should remain as default.

Create a security group for the machine on which WordPress will be installed. Select the created VPC and enter the inbound rules.

SSH ===============> Source: NAT Instance
HTTP ==============> Anywhere-IPv4
HTTPS =============> Anywhere-IPv4
ICMP IPv4 =============> Anywhere-IPv4 (opened to determine whether we are connected to the internet by pinging. It can be removed optionally.)
Outbound should remain as default.

Note: A database is required for WordPress to work, so RDS must be installed.

RDS Security Group: Select the created VPC and enter the inbound rules.

MySQL/Aurora ============= > Source: WordPress Security Group
Outbound should remain as default.

Let's create a security group for ALB to receive http requests from the internet. Select the created VPC and enter the inbound rules.

HTTP ===============> Anywhere-IPv4
HTTPS ===============> Anywhere-IPv4
Outbound should remain as default.

Once the security group creation process is complete, let's start creating the machines.

Let's start by creating the NAT instance:

To create a NAT instance via CLI, the required command is:

aws ec2 run-instances --image-id ami-005f9685cb30f234b --instance-type t2.micro --key-name <value> --security-group-ids <value> --subnet-id <value> --user -data file:/<value>

Create a standard EC2 machine with the selected Security Group and add the following commands to User Data. This machine will act as the NAT instance. In the Network settings, select Edit and choose the created VPC and Security Group.

User-data;
,,,,,
#! /bin/bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s "IP range specified in the project" -j MASQUERADE
,,,,,

After creating the Nat instance, select the machine Actions => Networking => Change source/destination => Stop packet control and save.
Our Wordpress machine requires a Nat instance to connect to the internet and serve as a bastion host/jump box.
After creating the Nat instance, select the Private Route table under Route Tables in VPC and select Routes => Edit Routes => Destination 0.0.0.0/0 => Select Target Instance and select the Nat instance.

Let's create an RDS Subnet;

Under the Subnet Group tab, create a Subnet group with at least 2 AZs and 2 Private subnets selected.

We can move on to RDS installation,

RDS :

-Select Standard and proceed,
-Select Mysql as the database it will run on.
-Select an appropriate template.
-Set the Username and Password in Credetentials Settings,
-In the Instance configuration, select the appropriate machine,
-Make an optional gb selection for Storage.
-For Connectivity, keep "Don’t connect to an EC2 compute resource" selected and select the created VPC and specify the subnet group. Public Access should be turned off, select the created Security Group, and make sure that Port 3306 is open.
-Under Additional configuration, give the first database name. Backup, Encryption, and Maintenance should be selected optionally.
-Let's create an RDS database and move on to installing the Wordpress machine.

Instance for Installing Wordpress;

The necessary command for creating through CLI;

aws ec2 run-instances --image-id ami-005f9685cb30f234b --instance-type t2.micro --key-name <value> --security-group-ids <value> --subnet-id <value> --user-data file:/<value>

Create a standard Ec2 machine and select the Security Group we created for Wordpress. Add the following commands to the User Data. This machine will serve as the instance where our website will be published. Select the created VPC and Security Group from Edit in Network settings.

Script to be added to User Data;
,,,,,
#!/bin/bash
db_username=admin
db_user_password=123456789
db_name=wordpress
db_user_host=RDS endpoint
yum update -y
amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
yum install -y httpd
systemctl start httpd
systemctl enable httpd
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
cd /var/www/html/
cp wp-config-sample.php wp-config.php
chown -R apache /var/www
chgrp -R apache /var/www
chmod 775 /var/www
find /var/www -type d -exec sudo chmod 2775 {} ;
find /var/www -type f -exec sudo chmod 0664 {} ;
sed -i "s/database_name_here/$db_name/g" wp-config.php
sed -i "s/username_here/$db_username/g" wp-config.php
sed -i "s/password_here/$db_user_password/g" wp-config.php
sed -i "s/localhost/$db_user_host/g" wp-config.php
systemctl restart httpd
,,,,,

Add the Nat instance we used as a jump box to the Route Table for the Wordpress machine to access the internet.
Routes => Edit Routes => Destination 0.0.0.0/0 => Select Target Instance and select the Nat instance.

Let's create a Target Group after creating our WordPress machine,

Target group;

Let's choose the Target Type as Instance,
Give a name to the target.
We can set Protocol and Port, it can remain default if desired.
Protocol version can be selected as HTTP1.
Health checks can be set as desired.
Let's add the WordPress instance to this target group.

After the Target Groups are finished, let's create a Load Balancer.

Load Balancer:

Let's start by selecting the Load Balancer type:
We will continue as ALB (Application Load Balancer),
Give a name to the Load Balancer,
Let's continue with the default settings.
Select VPC and 2 Public Subnets for 2 AZs.
Select the Security Group created for the ALB,
Under Listeners and routing, select the Target Group created.
Let's create it.

Get the ALB DNS Address and paste it into your web browser.

CONGRATULATIONS, YOUR SITE IS READY TO USE.