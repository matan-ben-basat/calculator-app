pipeline {
    agent {
        docker { 
            image 'python:3.9-slim'
            // השורה הזו אומרת לדוקר לרוץ כמשתמש על (root) וכך לפתור את בעיית ההרשאות!
            args '-u root:root'
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
                sh 'cd calculator-app-v2 && pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running tests inside ZIP folder...'
                sh 'cd calculator-app-v2 && python -m unittest discover -s tests -v'
            }
        }
    }
}
