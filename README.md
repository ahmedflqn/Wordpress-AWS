## **Hosting WordPress on AWS Cloud**
**In this guide, We will Create a WordPress architecture that is secure, highly available, fault-tolerant and also scalable.**

### 1- Creating our architecture diagram
![Architecture diagram
](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/WordPress%20architecture%20diagram%20(1).png?raw=true)

**Architecture components**

 - Cloudfront to cache closer to end user for faster delivery
 - Route 53 to register our  domain name or host existing one. It allows us to create some aliases that bind together our AWS resources so we will be able to register our domain name with Cloudfront 
 - Web instances in private subnets for maximum security, we will use Session Manager to connect to them instead of creating a bastion host.
 instances will run in an auto-scaling group for scalability with an elastic load balancer to distribute traffic and ensure high availability
 - Amazon RDS to host WordPress  database with Multi-AZ deployment. It will be deployed in a private subnet for security and we will allow access only from our instance security group.
 - Internet Gateway to enable communication between resources in VPC and the internet.
 - NAT Gateway in each availability Zone to enable EC2  instances in a private subnet to access the internet
 - Amazon EFS file system so EC2 can access shared WordPress data via EFS Mount Target in every availability Zone.
---
### 2- Create our VPC using AWS Cloudformation
This  template  deploys  a  VPC,  with  2  public  and 4 private  subnets  spread  across  two    Availability  Zones.  It  deploys  an  internet  gateway,  with  a  default  route  on  the  public  subnets.  It  deploys  a  pair  of  NAT  gateways  (one  in  each  AZ),  and  default  routes  for  them  in  the  private  subnets. Creates security groups for our App, EFS, and RDS instances
and an EC2 instance role.

[Cloudformation Template](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Cloudformation_WordPress.yml)
  
---
### 3- Creating our RDS database

   Before we create our RDS we need to create our RDS Subnet Group
 ![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/RDS-subnetgroup.PNG?raw=true)

   Now let's create our RDS
    Here we will use MySQL

![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/DB1.PNG?raw=true)

   We will choose the free tier to keep our costs low because this is a guide,
   But in the production environment, we would pick production and Multi-AZ deployment

   
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/DB2.PNG?raw=true)
   
 
 Make sure to save your database credentials
 
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/DB3.PNG?raw=true)

   Here we will choose the security group we created earlier for the RDS with Cloudformation. 
    Also, make sure public access is set to No
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/db22.PNG?raw=true)
### 3- Creating our EFS

   Move to the EFS Console 
   Click on Create file system
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/EFS1.PNG?raw=true)

   Make sure you mount EFS to the Private WordPress APP subnets and choose our WP-EFS-S.G Security Group
     then Press Next and Create!
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/EFS2.PNG?raw=true)
      

---
### 4-Create our launch template

   Move to the EC2 console under instances and choose the launch template we created earlier 
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/launchtemplate1.PNG?raw=true)
      
 Here we will choose t2.micro because it's included in the free tier.
 we will not need a key pair as we will connect to the instance using Session Manager 
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/launchtemplate2.PNG?raw=true)

   In advanced details we will need first to pick our EC2 WP instances profile 
         that we created earlier from the CloudFormation template. 
         IAM roles created allow us to connect our ec2 through Session Manager, 
         and that will be the only way because we didn't create a bastion host.
         We will allow our instance to connect to EFS and have full access.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/Launchtemplate3.PNG?raw=true)
         
   In user data, We will add the commands that automate the  
           preparation of our ec2 instance in order to host our WordPress website.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/Launchtemplate4.PNG?raw=true)

**User data script**
```
 #!/bin/bash -xe
sudo yum update -y
sudo yum install php -y
sudo yum install php-mysqli -y
sudo yum install mariadb105 -y

sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm

sudo usermod -a -G apache ec2-user  
sudo chown -R apache:apache /var/www/html
sudo chmod 2775 /var/www/html
sudo find /var/www/html -type d -exec chmod 2775 {} \;
sudo find /var/www/html -type f -exec chmod 0664 {} \;

sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0f629be95e3adfde5.efs.us-east-1.amazonaws.com:/ /var/www/html/

echo "fs-0f629be95e3adfde5.efs.us-east-1.amazonaws.com:/ /var/www/html/ nfs defaults,_netdev 0 0" >> /etc/fstab

wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz

```

**updating the instance and installing essential software, including Apache, PHP, and MySQL.**
```
sudo yum update -y
sudo yum install php -y
sudo yum install php-mysqli -y
sudo yum install mariadb105 -y
```

**We’ll activate these services and configure them to start automatically after a system restart.**
```
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```
**Fix permission on file system**
```
sudo usermod -a -G apache ec2-user  
sudo chown -R apache:apache /var/www
sudo chmod 2775 /var/www
sudo find /var/www -type d -exec chmod 2775 {} \;
sudo find /var/www -type f -exec chmod 0664 {} \;
```
**Mount our EFS file system and making sure it's automatically mounted after instance restart**
```
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0f629be95e3adfde5.efs.us-east-1.amazonaws.com:/ /var/www/html/

echo "fs-0f629be95e3adfde5.efs.us-east-1.amazonaws.com:/ /var/www/html/ nfs defaults,_netdev 0 0" >> /etc/fstab
```

**Download WordPress, unzip the downloaded file, and install it**
```
wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz
```


---

### 5-Create a target group for load balancer
Make sure you choose instances as the target type and HTTP as the listening protocol
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/Targetgroup1.PNG?raw=true)

Here don't register targets. We will add our autoscaling group later
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/targetgroup2.PNG?raw=true)

---
### 6-Create our load balancer
Before creating our load balancer. we need to create a load balancer Security group and allow HTTP, HTTPS
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/sg%20for%20loadbalancer.PNG?raw=true)

after that, we need to change our app security group to allow connection from the load balancer on HTTP
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/edit-app-sg.PNG?raw=true)

**Now let's create our load balancer**

In the EC2 dashboard go to load balancing and choose load balancer, 
and press on create load balancer

Here make sure to choose our public subnets PS1, PS2
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/lb1.PNG?raw=true)
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/loadbalancer2.PNG?raw=true)

### 7-Create our Autoscaling group
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/Autoscaling1.PNG?raw=true)

We will choose our app subnets
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/autoscaling2.PNG?raw=true)
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/autoscaling3.PNG?raw=true)

In group size here you can choose what you want for this settings. For now we leave it 1 to lower cost.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/asg1.PNG?raw=true)

After creating our auto-scaling group we will create our database for WordPress, and test our App through Load balancer DNS
first, we will need to connect to our ec2 Instance through Session Manager
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/session%20connect.PNG?raw=true)

  ```
  bash
  mysql -h "your database endpoint" -P 3306 -u admin -p   
  ```
  Enter database master password 
  ```
  create database Wordpress;
 ```
 to make sure we created the database
 ```
 show databases;
 ```  
               

![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/create%20database.PNG?raw=true)

Now let's test our app through ALB DNS
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/wp1.PNG?raw=true)
Our web App is running 
let's connect our database
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/wordpress%20database.PNG?raw=true)
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/wordpresstest1.PNG?raw=true)
Now after we check everything is fine let's complete our remaining steps.

---
### 8-Create CloudFront distribution

Choose your Load balancer 
Select HTTP as the default option for listener protocol.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/cloudfront1.PNG?raw=true)

For the Viewer setting, select HTTP and HTTPS, or you can choose to redirect HTTP to HTTPS. However, if you opt for the redirect option, you will need to request an SSL certificate.

Make sure to choose “GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE” as these methods allow the WordPress admin user to write, post, and configure the website.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/cloudfront2.PNG?raw=true)

![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/cloudfront3.PNG?raw=true)
Now we can access our web app through CloudFront Distribution domain name 

---
### 9-Register CloudFront distribution with route 53

Sign in to the AWS Management Console and open the Route 53 console.
Create a hosted zone in Route 53 for the domain you want to associate with your CloudFront distribution. 
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/hostedzone.PNG?raw=true)

Next in the Route 53 console, click on "Create Record Set" to create a new record set.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/hostedzone2.PNG?raw=true)
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/hostedzone4.PNG?raw=true)

For record name enter the domain name or subdomain you want to associate with the CloudFront distribution (e.g., project.wordpressonaws.com).
Type: Select "A - IPv4 address".
Alias: Select "Yes"
Alias Target: Click on the "Alias Target" field and select your CloudFront distribution from the dropdown list. It should be listed under the "CloudFront distributions" section.
Click on "Create" to save the record set.

---
### Now that was the final step.
We have completed all the steps required to host our WordPress App on AWS.
Insert your domain in your browser and create your WordPress Website.
![enter image description here](https://github.com/ahmedflqn/Wordpress-AWS/blob/main/Media/wp2.PNG?raw=true)
