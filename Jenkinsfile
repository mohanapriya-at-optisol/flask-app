pipeline {
    agent any

    environment {
        AWS_REGION   = "ap-south-1"
        INSTANCE_ID  = "i-0876c80ca0a7a4ff0"
        IMAGE_NAME   = "priyaobs/flask-demo:latest"
        TARGET_ROLE_ARN = credentials('target-role-arn')
    }

    stages {

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push ${IMAGE_NAME}
                      docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2 via SSM (Cross Account)') {
            steps {
                sh '''
set +x

# Assume target account role
read AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN <<< $(aws sts assume-role \
  --role-arn ${TARGET_ROLE_ARN} \
  --role-session-name jenkins-ssm \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY
export AWS_SESSION_TOKEN

# Deploy using SSM
aws ssm send-command \
  --region ${AWS_REGION} \
  --instance-ids ${INSTANCE_ID} \
  --document-name AWS-RunShellScript \
  --comment "Deploy Flask Docker App from Jenkins" \
  --parameters commands="[
    'docker pull ${IMAGE_NAME}',
    'docker stop flask-demo || true',
    'docker rm flask-demo || true',
    'docker run -d --name flask-demo -p 5000:5000 ${IMAGE_NAME}'
  ]"
'''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully"
        }
        failure {
            echo "❌ Deployment failed"
        }
    }
}
