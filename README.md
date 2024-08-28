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
•	Add your EC2 user to the Docker group to run Docker commands without **sudo**
```sh
sudo usermod -aG docker ec2-user
```
•	Install Docker Compose:
-	First, download the latest version of Docker Compose:
```sh
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
-	Then, apply executable permissions to the binary:
```sh
sudo chmod +x /usr/local/bin/docker-compose
```
•	Verify the installation of Docker and Docker Compose:
```sh
docker --version
docker-compose --version
```

3. Generate an SSH keypair from the ec2 instance using the following steps:
•	Run the following command on the EC2 instance:
```sh
ssh-keygen -t rsa
```
•	Add the Public Key to authorized_keys(To allow SSH connections using this key, add the public key to authorized_keys):
```sh
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
•	Copy the Private Key from the EC2 Instance to clipboard:
```sh
cat ~/.ssh/id_rsa
```
•	Copy the Public Key from the EC2 Instance to clipboard:
```sh
cat ~/.ssh/id_rsa
```

4. Navigate to your **Bitbucket Cloud account**, choose your exact **Repository** or create one if you do not have any.
Create a file called **docker-compose.yml** and paste in the following code which creates a Bitbucket datacenter container and attaches a database to it.
```sh
version: '3'
services:
  bitbucket:
    image: atlassian/bitbucket-server:latest  # Pull the latest Bitbucket Server image
    ports:
      - "7990:7990"
      - "7999:7999"
    volumes:
      - bitbucket-data:/var/atlassian/application-data/bitbucket
    environment:
      - ATL_JDBC_URL=jdbc:postgresql://db/bitbucketdb
      - ATL_JDBC_USER=bitbucketuser
      - ATL_JDBC_PASSWORD=bitbucketpassword

  db:
    image: postgres:9.6
    environment:
      - POSTGRES_DB=bitbucketdb
      - POSTGRES_USER=bitbucketuser
      - POSTGRES_PASSWORD=bitbucketpassword
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  bitbucket-data:
  db-data:
```

5. Navigate to the repository setting on your left pane, go to **SSH Keys**. Choose **use my own key**, then paste in the private and public keys you generated from your ec2 at the appropriate section, then **save the key**.  At the bottom, at the **Known hosts** section at the level of **Host address**, paste in your instance public_IP, hit on **fetch** and **add hosts**. You will have the following.

![image](https://github.com/user-attachments/assets/031a1bd6-9908-48b8-8f7b-c1b95081443d)

6. Go to the ‘Repository variable’ on the left pane. 
Add the following variables:
**SSH_PRIVATE_KEY** = add the private key you copied from your ec2
**EC2_USER** = the user in which you created your keypairs. For this demo, we used ‘ec2-user’
**AWS_ACCESS_KEY_ID** = your AWS access keys
**AWS_SECRET_ACCESS_KEY** = your AWS secret keys
**EC2_HOST** = your ec2’s public up or public DNS

![image](https://github.com/user-attachments/assets/d3c923f7-e26a-4911-a6a4-29baf81c469a)

8. Navigate to your Compass Cloud account, Click on **App** at the top.

![image](https://github.com/user-attachments/assets/c202c2e8-a7f6-41e0-9239-374fa7c889d7)

Search **Bitbucket**, click on it, Click “Get App” to install it.

![image](https://github.com/user-attachments/assets/1d83a610-be35-475c-a5ee-cb186977ba4a)

Click **Configure** to establish a connection with the exact repository of your bitbucket account.

![image](https://github.com/user-attachments/assets/8736c9af-2cd5-4035-9330-cb31ba7de81c)

Click **Connect**

![image](https://github.com/user-attachments/assets/6ba34606-ec26-4495-a8ed-abdc177b4da4)

Select the desired repository you wish to grant access to using the **drop-down** arrow then Click on **Grant Access**.

![image](https://github.com/user-attachments/assets/02d3f33c-f3ad-4fb6-bc57-efbd099cce27)

Navigate to your **Component** tab, click on **create**, Pass the name of the component, create your desired team then paste in the bitbucket URL in the provided space or choose the URL if options are given then hit on create.

![image](https://github.com/user-attachments/assets/9e5f268a-fefa-4ca1-8847-c827f276862b)

You will be navigated to the below interface to show that your connection with Bitbucket has been established.

![image](https://github.com/user-attachments/assets/79d7825d-28c9-426e-a61a-a09a5eb895fe)

9. In your Bitbucket account, click on **pipeline** on the left pane. 
Create your pipeline with the following script then commit the changes. **make sure to edit the region in your pipeline script to match the region in which your instance is running**
```sh
image: atlassian/default-image:latest

pipelines:
  default:
    - step:
        name: Deploy Docker Compose on EC2
        caches:
          - pip
        services:
          - docker
        script:
          # Install required tools in the Bitbucket pipeline environment
          - apt-get update && apt-get install -y openssh-client awscli

          # Setup AWS credentials in Bitbucket environment
          - mkdir -p ~/.aws
          - |
            echo "[default]" > ~/.aws/credentials
            echo "aws_access_key_id = $AWS_ACCESS_KEY_ID" >> ~/.aws/credentials
            echo "aws_secret_access_key = $AWS_SECRET_ACCESS_KEY" >> ~/.aws/credentials
            echo "[default]" > ~/.aws/config
            echo "region = eu-west-2" >> ~/.aws/config

          # Setup SSH access in Bitbucket environment
          - mkdir -p ~/.ssh
          - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          - chmod 600 ~/.ssh/id_rsa
          - ssh-keyscan -H $EC2_HOST >> ~/.ssh/known_hosts

          # Copy Docker Compose file to the EC2 instance's home directory
          - scp -i ~/.ssh/id_rsa docker-compose.yml $EC2_USER@$EC2_HOST:/home/$EC2_USER/docker-compose.yml

          # SSH into EC2 and run Docker Compose
          - |
            ssh -i ~/.ssh/id_rsa $EC2_USER@$EC2_HOST <<EOF
              # Ensure Docker is running
              if ! sudo systemctl is-active --quiet docker; then
                sudo systemctl start docker
              fi
              
              # Run Docker Compose using the file in the user's home directory
              docker-compose -f /home/$EC2_USER/docker-compose.yml up -d
            EOF

definitions:
  services:
    docker:
      memory: 2048
```

10. After committing the pipeline script, navigate back to **pipeline** to see how the script is deployed. After it deploys successfully, you will have the following view.

![image](https://github.com/user-attachments/assets/b52081f5-7afd-400f-8265-bb51caf9bd34)


11. Navigate to your AWS account, copy the public_I YEW p of your instance, paste in your browser then add in the relevant port **public_Ip:7990** or **public_Ip:7999**. You will have the following result.

![image](https://github.com/user-attachments/assets/3f4ab8b3-bb4f-4a43-b051-b9cfad4e3568)


12. Navigate to your compass account and you will see the below type of interface. You will see the below **dots** on your activity event display and if you click on it, it will redirect you to your **Bitbucket pipeline** deployment page. The dots depend on the number of your pipeline runs.


![image](https://github.com/user-attachments/assets/51a97a99-4ca7-47a6-b067-890eab237d1d)


**NB:** If you receive failures when running your pipeline script and after checking you get a **command not found** error after checking docker-compose version, on your Amazon Linux 2 instance, it might be due to one of the following reasons:

**Incorrect Installation Path**: Docker Compose might not be in your system's PATH.
**Installation Command Issue**: The command used to install Docker Compose may have failed or been incomplete.

To solve this:

•  Check if /usr/local/bin is in your PATH:
Run the following command to see your current PATH environment variable:
```sh
echo $PATH
```
If **/usr/local/bin** is not listed, you'll need to add it.

•  Add **/usr/local/bin** to your PATH:
You can temporarily add **/usr/local/bin** to your PATH with the following command:
```sh
export PATH=$PATH:/usr/local/bin
```

To make this change permanent, add it to your **.bashrc** or **.bash_profile**:
```sh
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

•  Check Docker Compose Version Again:
After adjusting the PATH, try checking the Docker Compose version:
```sh
docker-compose --version
```

**Then rerun the Bitbucket Pipeline**







