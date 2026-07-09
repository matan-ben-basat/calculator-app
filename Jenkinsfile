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
        PRODUCTION_IP  = "44.203.160.249" // ה-IP המעודכן והתקין של שרת הייצור[cite: 8]
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm[cite: 8]
            }
        }
        
        stage('Install System Tools & Deps') {
            steps {
                echo 'Installing AWS CLI and Docker CLI inside agent...'[cite: 8]
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
                '''[cite: 8]
                echo 'Installing requirements from ZIP folder...'[cite: 8]
                sh "cd ${APP_DIR} && pip install -r requirements.txt"[cite: 8]
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Running tests inside ZIP folder...'[cite: 8]
                sh "cd ${APP_DIR} && python -m unittest discover -s tests -v"[cite: 8]
            }
            post {
                success {
                    echo 'Archiving test results as artifacts...'[cite: 8]
                    archiveArtifacts artifacts: "${APP_DIR}/tests/*.py", allowEmptyArchive: true[cite: 8]
                }
            }
        }
        
        stage('Build & Push to ECR (CI Flow)') {
            when { changeRequest() }[cite: 8]
            steps {
                echo "CI Flow Triggered for PR #${env.CHANGE_ID}"[cite: 8]
                script {
                    def prTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"[cite: 8]
                    
                    echo "Logging into AWS ECR..."[cite: 8]
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"[cite: 8]
                    
                    echo "Building Docker Image..."[cite: 8]
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${prTag} ${APP_DIR}"[cite: 8]
                    
                    echo "Pushing Image to ECR..."[cite: 8]
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${prTag}"[cite: 8]
                }
            }
        }
        
        stage('Deploy & Verify (CD Flow)') {
            when { branch 'master' }[cite: 8]
            steps {
                echo "CD Flow Triggered: Promoting candidate image to production"[cite: 8]
                script {
                    def releaseTag = "build-${env.BUILD_NUMBER}"[cite: 8]
                    
                    echo "Logging into AWS ECR..."[cite: 8]
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"[cite: 8]
                    
                    echo "Building Production Candidate Images..."[cite: 8]
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${releaseTag} -t ${ECR_REGISTRY}/${ECR_REPO}:latest ${APP_DIR}"[cite: 8]
                    
                    echo "Pushing Production Images to ECR..."[cite: 8]
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${releaseTag}"[cite: 8]
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:latest"[cite: 8]
                    
                    echo "Deploying Automatically to Production EC2 via SSH..."
                    // שימוש במפתח המאובטח של ג'נקינס כדי להתחבר אוטומטית לשרת הפרודקשן ולהחליף את הקונטיינר
                    withCredentials([sshUserPrivateKey(credentialsId: 'prod-ec2-ssh-key', keyFileVariable: 'IDENTITY_KEY')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i ${IDENTITY_KEY} ubuntu@${PRODUCTION_IP} "
                                # התחברות ל-ECR מתוך שרת הפרודקשן כדי לאפשר משיכת אימג'ים[cite: 8]
                                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                
                                # משיכת האימג' העדכני ביותר שנדחף הרגע
                                docker pull ${ECR_REGISTRY}/${ECR_REPO}:latest
                                
                                # עצירה והסרה של הקונטיינר הישן כדי לפנות את פורט 5000[cite: 8]
                                docker rm -f calculator-prod || true
                                
                                # הרצת הקונטיינר החדש באופן אוטומטי ומפותח לפורט המארח[cite: 8]
                                docker run -d -p 5000:5000 --name calculator-prod --restart always ${ECR_REGISTRY}/${ECR_REPO}:latest
                            "
                        """
                    }
                    
                    echo "Executing Health Verification Probe..."[cite: 8]
                    sh """
                        retry=0
                        max_retries=5
                        # הוספת מגבלת זמן (Timeout) מונעת מהצינור לקפוא במקרה של עיכוב ברשת[cite: 8]
                        until [ \$(curl --connect-timeout 5 --max-time 10 -s -o /dev/null -w "%{http_code}" http://${PRODUCTION_IP}:5000/health) -eq 200 ] || [ \$retry -eq \$max_retries ]; do
                            echo "Application is not ready yet. Retrying in 10 seconds... (Attempt \$((\$retry + 1))/\$max_retries)"
                            sleep 10
                            retry=\$((\$retry + 1))
                        done
                        
                        if [ \$retry -eq \$max_retries ]; then
                            echo "ERROR: Health verification failed! Rolling back..."[cite: 8]
                            exit 1
                        fi
                        echo "Deployment successful! Service is healthy."[cite: 8]
                    """[cite: 8]
                }
            }
        }
    }
}
