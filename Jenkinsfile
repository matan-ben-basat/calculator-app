pipeline {
    agent {
        docker { 
            image 'python:3.9-slim'
            args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        AWS_ACCOUNT_ID = "992382545251"
        AWS_REGION     = "us-east-1"
        ECR_REGISTRY   = "992382545251.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPO       = "calculator-app-matan"
        APP_DIR        = "calculator-app-v2"
        PRODUCTION_IP  = "44.203.160.249" // ה-IP המעודכן והתקין של שרת הייצור
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }
        
        stage('Install System Tools & Deps') {
            steps {
                echo 'Installing AWS CLI and Docker CLI inside agent...'
                sh '''
                    apt-get update && apt-get install -y curl unzip
                    
                    # ניקוי שאריות מבילדים קודמים כדי למנוע תקיעה של unzip
                    rm -rf aws awscliv2.zip
                    
                    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                    # שימוש בדגל -o דורס קבצים אוטומטית ללא שאלות
                    unzip -o -q awscliv2.zip && ./aws/install
                    
                    # התקנת ה-Docker CLI
                    curl -fsSL https://get.docker.com -o get-docker.sh
                    sh get-docker.sh
                '''
                echo 'Installing requirements from ZIP folder...'
                sh "cd ${APP_DIR} && pip install -r requirements.txt"
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Running tests inside ZIP folder...'
                sh "cd ${APP_DIR} && python -m unittest discover -s tests -v"
            }
            post {
                success {
                    echo 'Archiving test results as artifacts...'
                    archiveArtifacts artifacts: "${APP_DIR}/tests/*.py", allowEmptyArchive: true
                }
            }
        }
        
        stage('Build & Push to ECR (CI Flow)') {
            when { changeRequest() } // כאן המיקום הנכון - לפני ה-steps!
            steps {
                echo "CI Flow Triggered for PR #${env.CHANGE_ID}"
                script {
                    def prTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    
                    echo "Logging into AWS ECR..."
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    
                    echo "Building Docker Image..."
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${prTag} ${APP_DIR}"
                    
                    echo "Pushing Image to ECR..."
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${prTag}"
                }
            }
        }
        
        stage('Deploy & Verify (CD Flow)') {
            when { branch 'master' } // כאן המיקום הנכון - לפני ה-steps!
            steps {
                echo "CD Flow Triggered: Promoting candidate image to production"
                script {
                    def releaseTag = "build-${env.BUILD_NUMBER}"
                    
                    echo "Logging into AWS ECR..."
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    
                    echo "Building Production Candidate Images..."
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${releaseTag} -t ${ECR_REGISTRY}/${ECR_REPO}:latest ${APP_DIR}"
                    
                    echo "Pushing Production Images to ECR..."
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${releaseTag}"
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:latest"
                    
                    echo "Deploying Automatically to Production EC2 via SSH..."
                    withCredentials([sshUserPrivateKey(credentialsId: 'prod-ec2-ssh-key', keyFileVariable: 'IDENTITY_KEY')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i ${IDENTITY_KEY} ubuntu@${PRODUCTION_IP} "
                                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                docker pull ${ECR_REGISTRY}/${ECR_REPO}:latest
                                docker rm -f calculator-prod || true
                                docker run -d -p 5000:5000 --name calculator-prod --restart always ${ECR_REGISTRY}/${ECR_REPO}:latest
                            "
                        """
                    }
                    
                    echo "Executing Health Verification Probe..."
                    sh """
                        retry=0
                        max_retries=5
                        until [ \$(curl --connect-timeout 5 --max-time 10 -s -o /dev/null -w "%{http_code}" http://${PRODUCTION_IP}:5000/health) -eq 200 ] || [ \$retry -eq \$max_retries ]; do
                            echo "Application is not ready yet. Retrying in 10 seconds... (Attempt \$((\$retry + 1))/\$max_retries)"
                            sleep 10
                            retry=\$((\$retry + 1))
                        done
                        
                        if [ \$retry -eq \$max_retries ]; then
                            echo "ERROR: Health verification failed! Rolling back..."
                            exit 1
                        fi
                        echo "Deployment successful! Service is healthy."
                    """
                }
            }
        }
    }
}
