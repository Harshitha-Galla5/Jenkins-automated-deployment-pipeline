pipeline {

    agent any

    tools {
        nodejs 'node 18'
    }

    environment {
        IMAGE_NAME = "devops-health-dashboard"
        EC2_HOST = credentials('ec2-host')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/Harshitha-Galla5/Jenkins-automated-deployment-pipeline.git'
                )
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build \
                -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push Docker Image') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo $DOCKER_PASS | docker login \
                    -u $DOCKER_USER \
                    --password-stdin

                    docker tag \
                    ${IMAGE_NAME}:${BUILD_NUMBER} \
                    $DOCKER_USER/${IMAGE_NAME}:${BUILD_NUMBER}

                    docker tag \
                    ${IMAGE_NAME}:${BUILD_NUMBER} \
                    $DOCKER_USER/${IMAGE_NAME}:latest

                    docker push \
                    $DOCKER_USER/${IMAGE_NAME}:${BUILD_NUMBER}

                    docker push \
                    $DOCKER_USER/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy Application') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sshagent(['ec2-ssh-key']) {

                        sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@$EC2_HOST "

                        echo $DOCKER_PASS | docker login \
                        -u $DOCKER_USER \
                        --password-stdin

                        docker pull $DOCKER_USER/${IMAGE_NAME}:latest

                        docker stop dashboard || true

                        docker rm dashboard || true

                        docker run -d \
                          --name dashboard \
                          --restart always \
                          -p 3000:3000 \
                          $DOCKER_USER/${IMAGE_NAME}:latest
                        "
                        '''
                    }
                }
            }
        }

        stage('Health Check') {

            steps {

                sh '''
                sleep 30

                curl -f http://${EC2_HOST}:3000/health
                '''
            }
        }
    }

    post {

        success {
            echo 'Application deployed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}
