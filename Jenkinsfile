pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'node-realworld'
        ACCOUNT_ID = '765309831951'  // replace with your AWS account ID
        IMAGE_TAG = "latest"
        SKIP_DB_TESTS = "true"       // <-- tells Jest to skip DB tests
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Tools') {
            steps {
                sh '''
                    # Ensure AWS CLI is available
                    if ! command -v aws &> /dev/null
                    then
                        echo "Installing AWS CLI..."
                        if command -v brew &> /dev/null; then
                            brew install awscli
                        else
                            pip3 install awscli --user
                            export PATH=/opt/homebrew/bin:$PATH
                        fi
                    fi

                    echo "AWS CLI version:"
                    aws --version || echo "⚠️ AWS CLI not found even after install"
                '''
            }
        }

        stage('Install & Test') {
            steps {
                sh '''
                    echo "📦 Installing dependencies..."
                    npm ci
                    echo "🧪 Running tests..."
                    export DATABASE_URL="file:./test.db"
                    npm test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "🐳 Building Docker image..."
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "🔑 Logging into ECR..."
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Create ECR Repo (if not exists)') {
            steps {
                sh '''
                    echo "📦 Ensuring ECR repository exists..."
                    aws ecr describe-repositories --repository-names $ECR_REPO --region $AWS_REGION >/dev/null 2>&1 || \
                    aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION
                '''
            }
        }

        stage('Tag & Push to ECR') {
            steps {
                sh '''
                    echo "🚀 Tagging and pushing Docker image to ECR..."
                    docker tag $ECR_REPO:$IMAGE_TAG $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                    docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Deploy Locally') {
            steps {
                sh '''
                    echo "🧩 Running Docker container locally for verification..."
                    docker ps -q --filter "name=node-realworld" | grep -q . && docker stop node-realworld || true
                    docker run -d -p 8080:3000 --name node-realworld $ECR_REPO:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully! Visit http://localhost:8080"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
