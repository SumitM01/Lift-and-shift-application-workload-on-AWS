# Lift-and-shift-application-workload-on-AWS
Lift the existing tech stack and host it on AWS cloud with proper load balancing and DNS configuration.

## Overview 
In this project i have hosted the entire stack of the vprofile project and hosted it no AWS for production workload. If we look at the scenario there are lot of services running in tandem so that the service properly delivered to the customer. Managing these services requires huge expenditure, time and resources.
To tackle this problem we can host the entire technology stack on AWS using its IaaS capabilities. We pay as we go, automation is easy, human errors can be avoided and flexibilty will be available.

## Project Architecture
![service_architecture](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/6edda220-61c6-40ee-abc1-1919cb75baaf)

## Services 
- EC2 instances
    - virtual machines to run tomcat, mysql, rabbitmq, memcached
- Elastic Load Balancer
    - Nginx load balancer replacement 
- Auto scaling
    - Automation for VM scaling
- S3/EFS storage
    - Shared storage
- Route 53
    - AWS private dns service
## Implementation

CONFIGURE SECURITY GROUPS :
	- configure vprofile elb sg
 ![vprofile-elb-sg rules](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/6f121752-0516-4b82-b3f8-edf3b386fbed)

	- configure vprofile-app-sg
 ![vprofile-app-sg rules](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/3ac3269a-bbcc-4d43-a4d2-fa76506d7162)

	- configure vprofile-backend-sg
 ![vprofile-backend-sg inbound rules](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/72122d80-2ce7-48e7-aafe-0a64ec5777de)

LOCAL MACHINE REPO
	- clone the repo vprofile-project and checkout to branch 'local-setup'
	- change to vagrant/automated_provisioning folder
	

LAUNCH EC2 INSTANCES AND CONFIGURE SERVICES
	- launch vprofile-db01 instance with centos 7 ami and t2.micro with user data as local-setup/vagrant/Automated_provisioning/mysql.sh and sg as vprofile-backend-sg with port 22 allowed from your ip
	- launch vprofile-mc01 instance with centos 7 ami and t2.micro with user data as local-setup/vagrant/Automated_provisioning/memcache.sh and sg as vprofile-backend-sg with port 22 allowed from your ip
	- launch vprofile-rmq01 instance with centos 7 ami and t2.micro with user data as local-setup/vagrant/Automated_provisioning/rabbitmq.sh and sg as vprofile-backend-sg with port 22 allowed from your ip
 ![backend-servers-ec2](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/4b02b966-52c2-4759-9bbc-610cb4864333)

	- ssh to backend instances and validate if they have been configured or not.
 ![rabbitmq-status-check](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/f96c6eeb-5bb9-43be-b5cd-8101318cf8e4)
 ![memcached-status-check-cli](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/285e7af7-0405-4a62-9cd1-e9fa4d723a52)
 ![mysql-status-check-cli](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/9afec310-63e0-4f1f-839c-5bef75812df2)
	- launch vprofile-app01 instance with ubuntu 20 ami and t2.micro with user data as local-setup/vagrant/Automated_provisioning/tomcat_ubuntu.sh and sg as vprofile-app-sg with port 22 allowed from your ip
	
NEW HOSTED ROUTE 53 ZONE
	- create a hosted zone in route 53 as vprofile.in
	- add records as db01.vprofile.in, mc01.vprofile.in, rmq01.vprofile.in with their respective private Ip addresses.
 ![hosted-zone-vprofile](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/24b0234a-99b0-4d92-9d88-e23e692d5b01)
 
UPDATE application.properties FILE IN 'aws-LiftAndShift'
	- update jdbc.url to jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
	- update memcached.active.host to mc01.vprofile.in
	- update rabbitmq.address to rmq01.vprofile.in

BUILD APPLICATION FROM SOURCE CODE
	- on local machine in the vprofile-project folder run mvn install artifact will be generated

MAKE S3 BUCKET AND STORE THE ARTIFACT IN THE BUCKET
	- create an IAM role for accessing S3 bucket through cli and setup aws cli
	- aws configure and provide the IAM role credentials
 ![iam-user-signin](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/47061ed8-05b7-4110-ac9b-f7aa78d0f602)
	- run 'aws s3 mb s3://your_bucket_name' to create a new bucket (make sure the bucket name is unique in the whole world)
	- go to your src code repo and run 'aws s3 cp vprofile-v2.war s3://your_bucket_name/vprofile-v2.war' to upload the artifact on the s3 bucket
 ![s3_artifact_upload](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/7b92ded8-8c8f-46d7-a520-8a48a92b3ded)

DOWNLOAD ARTIFACT ON APP SERVER AND BUILD AND RUN IT
	- create an ec2 role allowing s3fullaccess to download artifact from s3 bucket on tomcat instance
	- ssh to tomcat server instance, install and configure awscli
 ![update-iam-role-app01](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/b245a3ed-1e9e-4e4e-9890-b67927e8ab61)
	- run 'aws s3 cp s3://your_bucket_name/vprofile-v2.war /tmp/vprofile-v2.war' to download the file 
	- stop tomcat service
	- navigate to /var/lib/tomcat9/webapps and remove ROOT folder
	- copy the downloaded file from /tmp folder to that folder using 'cp /tmp/vprofile-v2.war ./ROOT.war'
	- start the tomcat service 

CONFIGURE LOAD BALANCER AND DNS
	- create a target group with 8080 port and health check at /login select vprofile-app01 as instance and reduce healthy threshold to 2.
 ![target-group](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/b349dc17-2eec-440e-a756-c9109da05de5)
	- create a load balancer as internet facing selecting all AZs and creating HTTPS listener with certificate
 ![load-balancer](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/b0affed3-01c0-45b3-92da-0565ac58eddf)
	- copy load balancer endpoint and create a cname record under hosted zones which routes to the endpoint
 ![add-cname-record](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/598c5055-9425-49cd-a60e-e7883bc4fe36)

CONFIGURE AUTOSCALING GROUP 
	- create an image of the app instance
	- create a launch configuration using the image of the instance
 ![launch-template](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/85040653-2eaa-4d25-aa8d-155a3ed00320)
	- create a auto scaling policy as you like
 ![auto-scaling-group](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/c7f89080-df68-442e-84ba-7ee9e910eee4)

## Results
![website-working](https://github.com/SumitM01/Lift-and-shift-application-workload-on-AWS/assets/65524561/deec4b58-5074-48c2-8fe4-682973ed21d6)
Congrats ! you have successfully migrated the entire stack and hosted your project on AWS. 

All the settings and configurations have been copied from the repository https://github.com/SumitM01/Automated-vprofile-project-setup-using-Vagrant. Hope this helps 😊.

# References
- https://github.com/devopshydclub/vprofile-project/tree/e4ba15005e3aee74516d725ef2db1bf3c419e472/vagrant/Automated_provisioning
- https://github.com/SumitM01/Automated-vprofile-project-setup-using-Vagrant
