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
                    docker login --username AWS --password-stdin 345485442601.dkr.ecr.ap-south-1.amazonaws.com
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
