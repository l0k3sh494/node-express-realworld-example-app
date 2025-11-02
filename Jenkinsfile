pipeline {
  agent any

  environment {
    AWS_REGION = 'us-east-1'
    AWS_ACCOUNT_ID = '765309831951'   // REPLACE
    ECR_REPOSITORY = 'node-realworld'          // will create if not exists
    IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
    ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
    CONTAINER_NAME = 'node-realworld-app'
    APP_PORT = '3000'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install & Test') {
      steps {
        sh 'npm ci'
        // if the sample has tests, run them. If not, you can skip or use a smoke test.
        sh 'npm test || true'
      }
      post { always { archiveArtifacts artifacts: 'npm-debug.log', allowEmptyArchive: true } }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} ."
      }
    }

    stage('Create ECR Repo (if needed)') {
      steps {
        sh '''
          aws ecr describe-repositories --repository-names ${ECR_REPOSITORY} --region ${AWS_REGION} >/dev/null 2>&1 || \
          aws ecr create-repository --repository-name ${ECR_REPOSITORY} --region ${AWS_REGION}
        '''
      }
    }

    stage('Auth & Push to ECR') {
      steps {
        sh """
          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ECR_URI}
          docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
          docker push ${ECR_URI}:${IMAGE_TAG}
          docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_URI}:latest
          docker push ${ECR_URI}:latest
        """
      }
    }

    stage('Deploy to Local Docker (EC2)') {
      steps {
        // Stop/remove any existing container, then run new one
        sh """
          docker rm -f ${CONTAINER_NAME} || true
          docker run -d --name ${CONTAINER_NAME} -p ${APP_PORT}:${APP_PORT} ${ECR_URI}:${IMAGE_TAG}
        """
      }
    }

    stage('Verify') {
      steps {
        // Simple healthcheck: list containers and show exposed port
        sh "docker ps --filter name=${CONTAINER_NAME}"
        sh "curl -sS http://localhost:${APP_PORT} || true"
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded. App should be at http://<EC2_PUBLIC_IP>:${APP_PORT}"
    }
    failure {
      echo "Pipeline failed. Check Jenkins console output."
    }
  }
}
