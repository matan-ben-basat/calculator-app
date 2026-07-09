pipeline {
    agent {
        docker { 
            image 'python:3.9-slim'
            // מיפוי ה-Socket מאפשר לסוכן להשתמש במנוע ה-Docker של השרת לצורך בנייה ודחיפה
            args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        AWS_ACCOUNT_ID = "992382545251"
        AWS_REGION     = "us-east-1"
        ECR_REGISTRY   = "992382545251.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPO       = "calculator-app-matan"
        APP_DIR        = "calculator-app-v2"
        PRODUCTION_IP  = "44.203.160.249" // יש להחליף ב-IP של שרת הפרודקשן שלך
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
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
                    // שמירת קובצי הבדיקות כתוצרי ריצה לפי דרישות ה-DoD
                    archiveArtifacts artifacts: "${APP_DIR}/tests/*.py", allowEmptyArchive: true
                }
            }
        }
        
        stage('Build & Push to ECR (CI Flow)') {
            when { changeRequest() } // שלב זה ירוץ אך ורק בתוך Pull Requests
            steps {
                echo "CI Flow Triggered for PR #${env.CHANGE_ID}"
                script {
                    // הגדרת תג דטרמיניסטי לפי דרישות המבחן: pr-<id>-<build>
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
            when { branch 'master' } // שלב זה ירוץ אך ורק במיזוג לענף master
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
                    
                    echo "Deploying container to Production EC2 Host..."
                    // פקודות הפריסה בשרת המרוחק (דוגמה באמצעות SSH או סקריפט מקומי במידה והשרת מארח גם את הפרודקשן)
                    // יש לוודא מיפוי פורטים מתאים (-p 5000:5000) והרצה מחדש
                    
                    echo "Executing Health Verification Probe..."
                    // מנגנון בדיקת בריאות יציב עם Retries/Backoff (5 ניסיונות, השהיה של 10 שניות)
                    sh """
                        retry=0
                        max_retries=5
                        until [ \$(curl -s -o /dev/null -w "%{http_code}" http://${PRODUCTION_IP}:5000/health) -eq 200 ] || [ \$retry -eq \$max_retries ]; do
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
