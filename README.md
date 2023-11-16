# AWS_Project

Project: AWS Three Tier Web Architecture

By,
Karthik NP
DevOps Engineer

Architecture overview:

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/5516f017-ce33-4d48-ac3c-4aa42b056f32)

                                         
In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.

Step.1: Downloading the Code from GitHub
I downloaded the code from AWS repo into my local environment by running the command below.
 $ git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/d3f0476d-1259-44af-8502-0e8ff0b6a3a5)


Step.2: S3 Bucket Creation
  Then I created an S3 bucket, with all the defaults configurations as in. This bucket is where I will upload my code later.

  ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/dad293cd-480a-4a5a-9612-05e3eac5049e)

                                                            
Step.3: IAM EC2 Instance Role Creation
•	Created an EC2 role including the following AWS managed policies. These policies will allow our instances to download our code from S3 and use Systems Manager Session Manager to securely connect to our instances without SSH keys through the AWS console.
1. AmazonSSMManagedInstanceCore                         2. 

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/3df2fff3-4b93-4e57-85df-98770909afd4)

 

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

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/8da1bc31-b5b8-4743-9ceb-0ab106360586)


Step.5: Database Deployment
Subnet Groups:
Added the subnets that created in each availability zone specifically for our database layer.
Database Deployment:
I selected a Standard create for this MySQL-Compatible Amazon Aurora database. I chose Dev/Test as a template since this isn't being used for production at the moment. I set a username and password to access our database. Next, under Availability and durability, I changed the option to create an Aurora Replica or reader node in a different availability zone. I set the security group that created for the database layer.       

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/3dcc3507-9433-4f1d-b6e1-196b22f21a9b)

Step.6: App Tier Instance Deployment
I created an EC2 instance for our app layer. The app layer consists of a Node.js application that will run on port 4000. I chose Amazon Linux 2 AMI with t.2 micro instance type and select the correct VPC, subnet, and IAM role. Since this is the app layer, I used one of the private subnets. Then selected a security group created for our private app layer instances. Since I’m using Systems Manager Session Manager to connect to the instance, proceeded without a keypair.


 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/e62d65a1-7382-4971-93c1-c058d38233c2)

	I connected my instance through the Session Manager. Switched to ec2-user by executing the following command:
$ sudo -su ec2-user
	To make sure that I’m able to reach the internet via our NAT gateways:
$ ping 8.8.8.8

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/3fdd80a4-9001-4043-a64b-0f5d6f070bcd)

Configuring the Database:
	Download the MySQL CLI:
$ sudo yum install mysql -y

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/9910b482-2e3a-4b54-953b-4383218b37d3)
        
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

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/5bc3919f-234c-4807-9ae9-738265156742)

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

"This is the health check"
	Then I tested our database connection by hitting the following endpoint locally:
curl http://localhost:4000/transaction
I got the response containing the test data I added earlier:
{"result":[{"id":1,"amount":400,"description":"groceries"},{"id":2,"amount":100,"description":"class"},{"id":3,"amount":200,"description":"other groceries"},{"id":4,"amount":10,"description":"brownies"}]}

Step.7: Internal Load Balancing and Auto Scaling
I created an Amazon Machine Image (AMI) of the app tier instance I created, and use that to set up autoscaling with a load balancer in order to make this tier highly available.
App Tier AMI:
I created an Image of the app-tier instance.

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/7f3ecd90-d9fd-4c2d-9fdc-167401147d08)

Target Group:
Then I create a target group to use with the load balancer.  The purpose of forming this target group is to use with our load balancer so it may balance traffic across our private app tier instances. Then I set the protocol to HTTP and the port to 4000, this is the port our Node.js app is running on. Changed the health check path to be /health. This is the health check endpoint of our app. I did not register any targets for now.

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/cd13b3b1-0809-4622-8fc8-56b366f77d3c)

Internal Load Balancer:
I created an Application Load Balancer for our HTTP traffic and selected internal since this one will not be public facing, but rather it will route traffic from our web tier to the app tier. Then selected the VPC and private subnets. 
Then selected the security group I created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our target group that I just created. 

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/93d7a135-f646-4511-8c0a-9611f8dff6f3)

Launch Template: I created a Launch template with the AMI I created earlier with the app tier created. For Key pair and Network Settings, I didn’t include it in the template. Since we don't need a key pair to access our instances and we'll be setting the network information in the autoscaling group. Then I used the security group for our app tier, and the same IAM role I have been using for our EC2 instances. 

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/c7c93f4a-3d55-46a8-9f35-f1ccd30c5619)

Auto Scaling:
I now created the Auto Scaling Group for our app instances by setting my VPC, and the private instance subnets for the app tier. Then attached this Auto Scaling Group to the Load Balancer I just created by selecting the existing load balancer's target group. I set desired, minimum and maximum capacity to 2. 
I now have our internal load balancer and autoscaling group configured correctly.

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/f8acfcac-2efc-4bd0-be34-91ec0d0f8711)

Step.8: Web Tier Instance Deployment: I deployed an EC2 instance for the web tier and made all necessary software configurations for the NGINX web server and React.js website.
Updating the Config File:
In the line 58 of the application-code/nginx.conf file I replaced [INTERNAL-LOADBALANCER-DNS] with my internal load balancer’s DNS entry. Then, uploaded this file and the application-code/web-tier folder to my S3 bucket.

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/578158a3-873f-4dd7-81b1-fbe2e5a5cdf6)

Web Instance Deployment:
I placed this web-tier instance in one of our public subnets. Then selected the correct network components, security group, and IAM role. I enabled the, auto-assign a public ip. And then proceeded without a key pair for this instance.

Connecting to Instance:
$ sudo -su ec2-user 
$ ping 8.8.8.8

Configure Web Instance:
	I installed all of the necessary components needed to run our front-end application. Again, started by installing NVM and node :
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
$ source ~/.bashrc
$ nvm install 16
$ nvm use 16

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/41afaa5b-3547-44eb-aa55-033adb9afc80)
          
	Now I downloaded our web tier code from my S3 bucket:
$ cd ~/
$ aws s3 cp s3://bucket-3-tier/web-tier/ web-tier –recursive
	Entered into the web-layer folder and created the build folder for the react app so I can serve our code:
$ cd ~/web-tier
$ npm install 
$ npm run build

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/c8555832-0457-4a6c-ba4a-8cc8ed383368)

NGINX can be used for different use cases like load balancing, content caching etc, but I’ll be using it as a web server that I will configure to serve our application on port 80, as well as help direct our API calls to the internal load balancer.
$ sudo amazon-linux-extras install nginx1 -y

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/4153dc01-534b-4850-ba31-b28f2480d81e)

	I now configured NGINX by going to the Nginx configuration file,
$ cd /etc/nginx
$ ls
	I deleted this nginx.conf file and used the one I uploaded to S3. 
$ sudo rm nginx.conf
$ sudo aws s3 cp s3:// bucket-3-tier/web-tier /nginx.conf .
	Then, restarted Nginx:
$ sudo service nginx restart
	To make sure Nginx has permission to access our files:
$ chmod -R 755 /home/ec2-user
	And then to make sure the service starts on boot:
$  chkconfig nginx on
Now when I plug in the public IP of our web tier instance, I should see my website and in that I can also see the database working.

Step.9: External Load Balancer and Auto Scaling:
I created an Amazon Machine Image (AMI) of the web tier instance and used that to set up autoscaling with an external facing load balancer in order to make this tier highly available.

Web Tier AMI:
I created an Image of the web-tier instance.

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/93fb95b6-1233-4975-94c8-fa0726735536)

Target Group:
I created our target group to use with the load balancer. The purpose of forming this target group is to use with our load balancer so it may balance traffic across our public web tier instances.

Then, I set the protocol to HTTP and the port to 80, this is the port NGINX is listening on, then changed the health check path to be /health. I did not register any targets for now.
 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/e8cb5a5d-7c70-40ce-b36d-406e16460737)

Internet Facing Load Balancer: I created an Application Load Balancer for our HTTP traffic and selected internet facing since this one will not be public facing, but rather it will route traffic from our web tier to the app tier. Then selected the VPC and public subnets.
Then selected the security group I created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our target group that I just created. 

 ![image](https://github.com/npk-aws/AWS_Project/assets/133031060/72c66158-57cb-4050-ad9c-0e2fb714e74e)

Launch Template:
I created a Launch template with the AMI I created earlier with the app tier. For Key pair and Network Settings, I didn’t include it in the template. Since we don't need a key pair to access our instances and we'll be setting the network information in the autoscaling group. Then I used the security group for our web tier, and the same IAM role I have been using for our EC2 instances.

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/c59f68c7-31d1-42e3-bd96-33e5c0ce8e6e)

Auto Scaling:
I now created the Auto Scaling Group for our web instances by setting my VPC, and the public subnets for the web tier. Then attached this Auto Scaling Group to the Load Balancer I just created by selecting the existing web tier load balancer's target group. I set desired, minimum and maximum capacity to 2. 
I now have our external load balancer and autoscaling group configured correctly.
I should see the autoscaling group spinning up 2 new web tier instances. To test if this is working correctly, I deletde one of our new instances manually and waited to see if a new instance is booted up to replace it. 
To test if my entire architecture is working, I navigated to our external facing load balancer, and plugged in the DNS name into my browser.

![image](https://github.com/npk-aws/AWS_Project/assets/133031060/129cbd6d-e2cc-4b5b-b64d-101007473428)

