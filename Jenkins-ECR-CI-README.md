# GitHub → Jenkins → Docker → Amazon ECR CI Pipeline

A hands-on DevOps project demonstrating an automated Continuous
Integration (CI) pipeline using **GitHub, Jenkins, Docker, Amazon EC2,
Amazon ECR, AWS IAM, and GitHub Webhooks**.

The pipeline automatically starts when code is pushed to GitHub, checks
out the source code, builds a Docker image, authenticates with Amazon
ECR, tags the image, and pushes it to a private ECR repository.

------------------------------------------------------------------------

## Project Overview

### Objective

Build an automated CI pipeline:

``` text
Developer Push
      ↓
    GitHub
      ↓
 GitHub Webhook
      ↓
    Jenkins
      ↓
 Git Checkout
      ↓
 Docker Build
      ↓
  ECR Login
      ↓
 Docker Tag
      ↓
 Docker Push
      ↓
 Amazon ECR
```

### Technologies

-   GitHub
-   Jenkins
-   Jenkins Pipeline
-   Git
-   Docker
-   Amazon EC2
-   Amazon ECR
-   AWS IAM
-   AWS CLI
-   Linux
-   GitHub Webhooks

------------------------------------------------------------------------

# 1. Project Architecture

Amazon EC2 is used as the Jenkins server.

``` text
                         ┌──────────────────┐
                         │      GitHub      │
                         │  Jenkins-ECR-CI  │
                         └────────┬─────────┘
                                  │
                            Push Webhook
                                  │
                                  ▼
                         ┌──────────────────┐
                         │     Jenkins      │
                         │   Amazon EC2     │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
             Checkout Source              Docker Build
                    │                           │
                    └─────────────┬─────────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │   Amazon ECR     │
                         │ Private Registry │
                         └──────────────────┘
```

------------------------------------------------------------------------

# 2. GitHub Repository

Repository:

``` text
https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git
```

The repository contains the application source and Dockerfile.

Example structure:

``` text
Jenkins-ECR-CI/
├── Dockerfile
└── index.html
```

## Dockerfile

``` dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

## Application

`index.html` is copied into the Nginx web server image during the Docker
build.

------------------------------------------------------------------------

# 3. Amazon ECR Repository

A private Amazon ECR repository was created:

``` text
hands-on/jenkins-ecr
```

Repository URI:

``` text
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr
```

The pipeline pushes:

``` text
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

------------------------------------------------------------------------

# 4. Amazon EC2 Setup

An Amazon Linux 2023 EC2 instance was used as the Jenkins server.

Instance type:

``` text
t2.micro
```

The instance hosts:

-   Jenkins
-   Docker
-   Git
-   Java 21
-   AWS CLI

Jenkins was accessed through:

``` text
http://<EC2-PUBLIC-IP>:8080
```

> For this learning environment, Jenkins was exposed on port 8080. A
> production setup should use HTTPS on port 443 through a reverse proxy
> or load balancer and keep Jenkins port 8080 private.

------------------------------------------------------------------------

# 5. IAM Role

An IAM role was attached to the EC2 instance:

``` text
JenkinsEC2-ECR-Role
```

The role provides Jenkins with access to Amazon ECR.

During the hands-on exercise, an AWS-managed ECR policy was used to
simplify the initial setup.

### Security improvement

For production, replace broad ECR permissions with a custom
**least-privilege IAM policy** containing only the ECR actions required
by the pipeline.

------------------------------------------------------------------------

# 6. Docker Installation and Jenkins Access

Docker was installed and started on Amazon Linux 2023.

The Docker service was enabled and verified.

Because Jenkins runs under the `jenkins` Linux user, Jenkins needed
access to the Docker daemon.

The following command was used:

``` bash
sudo usermod -aG docker jenkins
```

Jenkins was restarted:

``` bash
sudo systemctl restart jenkins
```

Docker access was verified with:

``` bash
sudo -u jenkins docker ps
```

This resolved the Docker socket permission error:

``` text
permission denied while trying to connect to the Docker daemon socket
```

------------------------------------------------------------------------

# 7. Java and Jenkins

Jenkins required a supported Java runtime, so Java 21 was installed
using Amazon Corretto:

``` text
java-21-amazon-corretto
```

Jenkins was then installed and started as a systemd service.

Status was verified with:

``` bash
sudo systemctl status jenkins
```

Expected:

``` text
Active: active (running)
```

Jenkins was accessed through:

``` text
http://<EC2-PUBLIC-IP>:8080
```

------------------------------------------------------------------------

# 8. Jenkins Pipeline Job

A Pipeline job was created:

``` text
jenkins-docker-ecr-pipeline
```

The final Pipeline script is:

``` groovy
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t nginx:latest .'
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                aws ecr get-login-password --region ap-south-1 | \
                docker login --username AWS --password-stdin \
                345485442601.dkr.ecr.ap-south-1.amazonaws.com
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                docker tag nginx:latest \
                345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                docker push \
                345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
                '''
            }
        }
    }
}
```

------------------------------------------------------------------------

# 9. Pipeline Stages Explained

## Stage 1 --- Checkout

``` groovy
git branch: 'main',
    url: 'https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git'
```

Jenkins clones the repository and checks out the `main` branch.

### Why specify `main`?

The repository uses `main`.

The initial simplified checkout was:

``` groovy
git 'https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git'
```

Jenkins attempted to resolve `master`, producing:

``` text
Couldn't find any revision to build.
```

Explicitly specifying `main` fixed the problem.

------------------------------------------------------------------------

## Stage 2 --- Docker Build

``` groovy
docker build -t nginx:latest .
```

Jenkins builds the Docker image using the Dockerfile from the
checked-out workspace.

The local image is:

``` text
nginx:latest
```

The `.` means the current Jenkins workspace is used as the Docker build
context.

------------------------------------------------------------------------

## Stage 3 --- ECR Login

``` bash
aws ecr get-login-password --region ap-south-1 |
docker login --username AWS --password-stdin \
345485442601.dkr.ecr.ap-south-1.amazonaws.com
```

This obtains an ECR authentication token and authenticates Docker
against the ECR registry.

Expected:

``` text
Login Succeeded
```

The EC2 IAM role supplies AWS credentials to the AWS CLI without storing
long-lived access keys on the server.

------------------------------------------------------------------------

## Stage 4 --- Tag Image

The local image:

``` text
nginx:latest
```

is given an ECR-compatible tag:

``` bash
docker tag nginx:latest \
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

This does not rebuild the image. The two tags reference the same image.

``` text
nginx:latest

345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

The ECR-formatted tag tells Docker where the image should be pushed.

------------------------------------------------------------------------

## Stage 5 --- Push to ECR

``` bash
docker push \
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

Docker uploads the required image layers and manifest to the private ECR
repository.

The image can then be viewed under:

``` text
AWS Console
→ ECR
→ Repositories
→ hands-on/jenkins-ecr
```

------------------------------------------------------------------------

# 10. GitHub Webhook Automation

After proving the pipeline worked manually, automatic triggering was
enabled.

In Jenkins:

``` text
Job
→ Configure
→ Build Triggers
→ GitHub hook trigger for GITScm polling
```

The trigger was enabled.

------------------------------------------------------------------------

# 11. GitHub Webhook Configuration

In GitHub:

``` text
Repository
→ Settings
→ Webhooks
→ Add webhook
```

Payload URL:

``` text
http://<EC2-PUBLIC-IP>:8080/github-webhook/
```

Content type:

``` text
application/json
```

Events:

``` text
Just the push event
```

Webhook:

``` text
Active
```

GitHub successfully delivered the webhook, confirmed by the green
delivery indicator.

------------------------------------------------------------------------

# 12. Why `/github-webhook/`?

Jenkins provides the:

``` text
/github-webhook/
```

endpoint for receiving GitHub webhook notifications.

Normal Jenkins UI:

``` text
http://<EC2-PUBLIC-IP>:8080/
```

GitHub webhook receiver:

``` text
http://<EC2-PUBLIC-IP>:8080/github-webhook/
```

The flow is:

``` text
GitHub Push
     ↓
POST /github-webhook/
     ↓
Jenkins receives event
     ↓
Jenkins triggers pipeline
     ↓
Jenkins checks out source
```

The webhook does not transfer the complete project to Jenkins. It
notifies Jenkins that a repository event occurred; Jenkins then performs
the Git checkout.

------------------------------------------------------------------------

# 13. Why GitHub Integration Is Not Required for Checkout

Jenkins can clone a GitHub repository directly using Git over HTTPS:

``` groovy
git branch: 'main',
    url: 'https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git'
```

GitHub webhook triggering is a separate function.

### Without webhook

``` text
Developer pushes
       ↓
GitHub
       ↓
Jenkins does not automatically start
       ↓
User clicks Build Now
```

### With webhook

``` text
Developer pushes
       ↓
GitHub
       ↓
Webhook
       ↓
Jenkins
       ↓
Automatic build
```

Therefore, Git checkout and webhook triggering solve different problems.

------------------------------------------------------------------------

# 14. End-to-End Test

A small change was made to `index.html` and committed to the `main`
branch.

GitHub sent the webhook.

Jenkins automatically started build #16.

Jenkins reported:

``` text
Started by GitHub push by powerstar7696-afk
```

All stages completed successfully:

``` text
Checkout      ✓
Docker Build  ✓
ECR Login     ✓
Tag Image     ✓
Push to ECR   ✓
```

This confirmed that the complete automated CI pipeline was working.

------------------------------------------------------------------------

# 15. Troubleshooting

## Problem 1 --- Jenkins executor waiting

### Symptom

``` text
Still waiting to schedule task
Waiting for next available executor
```

### Cause

The Jenkins built-in node was marked offline because the small EC2
instance's temporary filesystem did not meet Jenkins' default free-space
threshold.

The `t2.micro` instance had `/tmp` mounted as `tmpfs`, with less than
Jenkins' default 1 GiB temporary-space requirement.

### Solution

The Jenkins node monitor thresholds were adjusted to values appropriate
for the small lab instance.

------------------------------------------------------------------------

## Problem 2 --- `master` branch not found

### Error

``` text
Couldn't find any revision to build.
```

### Cause

The repository uses:

``` text
main
```

while the initial checkout attempted:

``` text
master
```

### Solution

Explicitly specify:

``` groovy
git branch: 'main',
    url: 'https://github.com/powerstar7696-afk/Jenkins-ECR-CI.git'
```

------------------------------------------------------------------------

## Problem 3 --- Jenkins could not access Docker

### Error

``` text
permission denied while trying to connect to the Docker daemon socket
```

### Cause

Docker worked for the EC2 user, but Jenkins runs as the `jenkins` Linux
user.

### Solution

``` bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Verify:

``` bash
sudo -u jenkins docker ps
```

------------------------------------------------------------------------

## Problem 4 --- ECR authentication denied

### Error

``` text
not authorized to perform:
ecr:GetAuthorizationToken
```

### Cause

The IAM role did not have the required ECR authorization permission at
that point.

### Solution

The ECR IAM permissions attached to the EC2 role were corrected.

The following command then succeeded:

``` bash
aws ecr get-login-password --region ap-south-1 |
docker login --username AWS --password-stdin \
345485442601.dkr.ecr.ap-south-1.amazonaws.com
```

------------------------------------------------------------------------

# 16. Security Considerations

The current project intentionally uses a simple setup for learning.

Jenkins is currently accessed through:

``` text
http://<EC2-PUBLIC-IP>:8080
```

This is not the preferred production architecture.

A more secure architecture is:

``` text
GitHub
   ↓
HTTPS :443
   ↓
Nginx / Application Load Balancer
   ↓
Jenkins :8080 (internal)
```

Recommended improvements:

-   Use HTTPS/TLS.
-   Do not expose Jenkins port 8080 publicly.
-   Restrict SSH port 22 to a trusted IP.
-   Use GitHub webhook signature/secret validation.
-   Use least-privilege IAM permissions.
-   Prefer IAM roles over long-lived AWS access keys.
-   Consider separate Jenkins agents for larger environments.

------------------------------------------------------------------------

# 17. Planned Secure Webhook Architecture

The next security exercise is to place Jenkins behind HTTPS.

``` text
                         HTTPS :443
GitHub ──────────────────────────────► Nginx / ALB
                                        │
                                        │ Internal
                                        ▼
                                  Jenkins :8080
```

Security Group example:

``` text
443 → 0.0.0.0/0
22  → Your IP only
8080 → No public access
```

The webhook would then use:

``` text
https://jenkins.example.com/github-webhook/
```

Additional protection can include GitHub webhook secret/signature
validation.

------------------------------------------------------------------------

# 18. Future Improvements

## Docker

-   Use an application-specific image name instead of `nginx:latest`.
-   Use immutable image tags.
-   Tag images with the Jenkins build number or Git commit SHA.
-   Add image vulnerability scanning.

Example:

``` text
jenkins-ecr:build-16
jenkins-ecr:<git-commit-sha>
```

## Jenkins

-   Store the Pipeline in a `Jenkinsfile` inside GitHub.
-   Use Jenkins credentials/environment variables where appropriate.
-   Add automated tests.
-   Add notifications.
-   Use separate build agents for scalable workloads.

## CI/CD

A future deployment pipeline could become:

``` text
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Checkout
   ↓
Unit Tests
   ↓
Docker Build
   ↓
Security Scan
   ↓
ECR Push
   ↓
Deployment
   ↓
AWS ECS / EKS / EC2
```

------------------------------------------------------------------------

# 19. Key DevOps Concepts Learned

This project demonstrates:

-   Git repository integration
-   Jenkins Pipeline
-   Jenkins executors
-   Linux users and groups
-   Docker daemon permissions
-   Docker image lifecycle
-   Docker tagging
-   Docker registry authentication
-   Amazon ECR
-   AWS IAM roles
-   AWS CLI
-   GitHub Webhooks
-   Automated CI triggers
-   CI troubleshooting
-   Cloud security fundamentals

------------------------------------------------------------------------

# 20. Useful Commands

### Docker

``` bash
docker --version
docker ps
docker images
```

### Jenkins

``` bash
sudo systemctl status jenkins
sudo systemctl restart jenkins
```

### Test Docker as Jenkins

``` bash
sudo -u jenkins docker ps
```

### Build

``` bash
docker build -t nginx:latest .
```

### Tag

``` bash
docker tag nginx:latest \
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

### ECR Login

``` bash
aws ecr get-login-password --region ap-south-1 |
docker login --username AWS --password-stdin \
345485442601.dkr.ecr.ap-south-1.amazonaws.com
```

### Push

``` bash
docker push \
345485442601.dkr.ecr.ap-south-1.amazonaws.com/hands-on/jenkins-ecr:latest
```

------------------------------------------------------------------------

# 21. Final Result

The project successfully achieved the intended objective:

> A GitHub push automatically triggers Jenkins. Jenkins checks out the
> source code, builds a Docker image, authenticates with Amazon ECR,
> tags the image, and pushes it to a private ECR repository.

Final status:

``` text
GitHub → Jenkins        ✅
Jenkins → Docker        ✅
Docker → ECR            ✅
Webhook automation      ✅
End-to-end CI pipeline  ✅
```

------------------------------------------------------------------------

## Author

**Uday Kiran Reddi**

DevOps / Cloud Engineering Hands-on Project

`AWS` `Jenkins` `Docker` `ECR` `GitHub` `Linux` `IAM` `CI/CD`
