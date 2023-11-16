# AWS_Project

Project: AWS Three Tier Web Architecture




By,
Karthik NP
DevOps Engineer

Architecture overview:
                                         
In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.

Step.1: Downloading the Code from GitHub
I downloaded the code from AWS repo into my local environment by running the command below.
 $ git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git
 

Step.2: S3 Bucket Creation
  Then I created an S3 bucket, with all the defaults configurations as in. This bucket is where I will upload my code later.
                                                            
Step.3: IAM EC2 Instance Role Creation
•	Created an EC2 role including the following AWS managed policies. These policies will allow our instances to download our code from S3 and use Systems Manager Session Manager to securely connect to our instances without SSH keys through the AWS console.
1. AmazonSSMManagedInstanceCore                         2. AmazonS3ReadOnlyAccess
 

Step.4: Networking and Security
I built out the VPC networking components as well as security groups that will add a layer of protection around our EC2 instances, Aurora databases, and Elastic Load Balancers.
•	Created an isolated network using the following components:
o	VPC
o	Subnets
o	Route Tables
o	Internet Gateway
o	NAT gateway
o	Security Groups
VPC Creation:
I have chosen a CIDR range that will allow us to create at least 6 subnets.
Subnet Creation:
We will need six subnets across two availability zones. Each subnet in one availability zone will correspond to one layer of our three-tier architecture.
Note: Our CIDR range for the subnets will be subsets of our VPC CIDR range.
Internet Connectivity:
•	Internet Gateway
In order to give the public subnets in my VPC internet access, I created and attached an Internet Gateway and attached it to my VPC.
•	NAT Gateway
In order for our instances in the app layer private subnet to be able to access the internet they will need to go through a NAT Gateway. For high availability, I deployed one NAT gateway in each of our public subnets and then allocated an Elastic IP for each.
Routing Configuration:
I created one route table for the web-layer public subnets. Added a route that directs traffic from the VPC to the internet gateway.
Then I changed the Subnet Associations by selecting the two web-layer public subnets I created earlier and created 2 more route tables, one for each app layer private subnet in each availability zone. These route tables will route app layer traffic destined for outside the VPC to the NAT gateway in the respective availability zone. Then add the subnet associations for each of the app layer private subnets.
Security Groups:
Security groups will tighten the rules around which traffic will be allowed to our Elastic Load Balancers and EC2 instances. 
The first security group I created is for the public, internet facing load balancer. Then added an inbound rule to allow HTTP type traffic for my IP.
The second security group I created is for the public instances in the web tier, then added an inbound rule that allows HTTP traffic from my internet facing load balancer security group. This will allow traffic from my public facing load balancer to hit our instances. Then, added an additional rule that will allow HTTP type traffic for my IP.
The third security group will be for my internal load balancer. Then added an inbound rule that allows HTTP type traffic from my public instance security group. This will allow traffic from our web tier instances to hit my internal load balancer.
The fourth security group I created is for our private instances. Then added an inbound rule that will allow TCP type traffic on port 4000 from the internal load balancer security group. This is the port our app tier application is running on and allows our internal load balancer to forward traffic on this port to our private instances. 
The fifth security group I created protects our private database instances. I added an inbound rule that will allow traffic from the private instance security group to the MYSQL/Aurora port (3306).
 

Step.5: Database Deployment
Subnet Groups:
Added the subnets that created in each availability zone specifically for our database layer.
Database Deployment:
I selected a Standard create for this MySQL-Compatible Amazon Aurora database. I chose Dev/Test as a template since this isn't being used for production at the moment. I set a username and password to access our database. Next, under Availability and durability, I changed the option to create an Aurora Replica or reader node in a different availability zone. I set the security group that created for the database layer.       
                                             
Step.6: App Tier Instance Deployment
I created an EC2 instance for our app layer. The app layer consists of a Node.js application that will run on port 4000. I chose Amazon Linux 2 AMI with t.2 micro instance type and select the correct VPC, subnet, and IAM role. Since this is the app layer, I used one of the private subnets. Then selected a security group created for our private app layer instances. Since I’m using Systems Manager Session Manager to connect to the instance, proceeded without a keypair.
 
	I connected my instance through the Session Manager. Switched to ec2-user by executing the following command:
$ sudo -su ec2-user
	To make sure that I’m able to reach the internet via our NAT gateways:
$ ping 8.8.8.8
       
Configuring the Database:
	Download the MySQL CLI:
$ sudo yum install mysql -y

        
	Initiating my DB connection with my Aurora RDS writer endpoint. In the following command, I replaced the RDS writer endpoint and the username, and then executed.
$ mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
$ mysql -h database-1-instance-1.coe1hc5e6okb.us-east-1.rds.amazonaws.com -u admin -p
Then I entered the password in the prompt and connected to my database. 
	Then created a database called webappdb with the following command using the MySQL CLI:
CREATE DATABASE webappdb;   
	To verify that it was created correctly:
SHOW DATABASES;
	Then created a data table by first navigating to the database we just created:
USE webappdb;    
	Then, created the following transactions table by executing this create table command:
CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL
AUTO_INCREMENT, amount DECIMAL(10,2), description
VARCHAR(100), PRIMARY KEY(id));    
	To verify the table was created:
SHOW TABLES;    
	Inserted some data into table for use/testing later:
INSERT INTO transactions (amount,description) VALUES ('400','groceries');   
	To verify that my data was added:
SELECT * FROM transactions;
Exited from the MySQL client.
Configuring the App Instance:
I updated the database credentials for the app tier by opening the application-code/app-tier/DbConfig.js file from the GitHub repo in VS Code. Filled the strings for the hostname, user, password and database with my credentials, the writer endpoint of my database as the hostname, and webappdb for the database and saved. Then uploaded the app-tier folder to my S3 bucket.
Now I installed all of the necessary components to run our backend application.
	 Started by installing NVM (node version manager).
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
$ source ~/.bashrc
	Next, I installed a compatible version of Node.js and made sure it's being used
$ nvm install 16
$ nvm use 16
	PM2 is a daemon process manager that will keep our node.js app running when we exit the instance or if it is rebooted. So I installed that as well.
$ npm install -g pm2   
	Now I downloaded our code from our S3 bucket onto our instance.
$ cd ~/
$ aws s3 cp s3://bucket-3-tier/app-tier/ app-tier –recursive
	Entered into the app directory, installed dependencies, and started the app with pm2.
$ cd ~/app-tier
$ npm install
$ pm2 start index.js
 
	To make sure the app is running correctly:
$ pm2 list 
	pm2 is just making sure our app stays running when I leave the SSM session. However, if the server is interrupted for some reason, we still want the app to start and keep running. This is also important for the AMI we will create later:
$ pm2 startup
	To setup the Startup Script, I pasted the command I got that started with 
$ sudo env PATH=$PATH:/home/ec2-user….
	To save the current list of node processes:
$ pm2 save

Testing App Tier:
	To hit out health check endpoint:
$ curl http://localhost:4000/health
	The response should look like this:
