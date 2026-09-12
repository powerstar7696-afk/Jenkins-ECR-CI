# Jenkins-ECR-CI

### GitHub → Jenkins → Docker build → Amazon ECR

 **Step 1 — Create the GitHub repository**
   -  Create a new GitHub repository
   -  Create a Dockerfile
   -  Create index.html

 **Step 2 — Create the Amazon ECR Repository**

 **Step 3 — prepare the Jenkins EC2 machine. 🚀**

 **Step 4 — Give EC2 permission to push to ECR**
  - Instead of storing AWS access keys inside Jenkins, we'll use an IAM Role attached to the EC2 instance. This is the better real-world approach.

 **Step 5 — Connect to EC2 and Install Docker**

 **Step 6 — Install Git and Jenkins**
  - sudo yum install git -y
  - sudo yum install java-21-amazon-corretto -y

  - sudo wget -O /etc/yum.repos.d/jenkins.repo \
https://pkg.jenkins.io/redhat-stable/jenkins.repo
  - sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2026.key
  - sudo yum install jenkins -y
