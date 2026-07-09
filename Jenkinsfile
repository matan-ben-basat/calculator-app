pipeline {
    agent {
        docker { 
            image 'python:3.9-slim' 
        }
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
                // פקודת cd אומרת לו להיכנס לתיקיית ה-ZIP ושם להריץ את ה-pip
                sh 'cd calculator-app-v2 && pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running tests inside ZIP folder...'
                // נכנסים לתיקיית ה-ZIP ומריצים את הטסטים המקוריים שהמרצים שמו שם
                sh 'cd calculator-app-v2 && python -m unittest discover -s tests -v'
            }
        }
    }
}
