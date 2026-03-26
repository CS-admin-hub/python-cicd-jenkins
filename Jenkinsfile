pipeline {
    agent { label 'ec2-agent' }

    environment {
        AWS_REGION = credentials('AWS_REGION_NAME')
        ACCOUNT_ID = credentials('AWS_ACCOUNT_ID')
        ECR_REPO   = "python-app-jenkins"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        IMAGE_URI  = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/CS-admin-hub/python-cicd-jenkins.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION \
                    | docker login --username AWS \
                      --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh 'docker tag $ECR_REPO:$IMAGE_TAG $IMAGE_URI'
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push $IMAGE_URI'
            }
        }
    }

    post {
        success { echo "✅ Image pushed successfully — Build #${BUILD_NUMBER}" }
        failure { echo "❌ Pipeline failed — check logs" }
    }
}
