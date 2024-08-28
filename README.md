# Deploying a Containerized Bitbucket Datacenter in AWS Ec2 Instance using Bitbucket Pipeline and Monitoring Deployment Activity using Compass

1. Create an ec2 instance in AWS with t2. Medium-instance type, AMI-Amazon Linux 2 and at the level of security group, allowing SSH from anywhere, also, allowing TCP traffic from Ports 7999 and 7990.

2. SSH into the instance and do the following:
•	Update the server, install docker, start and enable it
```sh
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo systemctl enable docker
```

4. 
