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
                echo 'Installing requirements...'
                sh 'pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running unit and integration tests...'
                // פקודת הרצת הטסטים המדויקת מה-README של התרגיל!
                sh 'python -m unittest discover -s tests -v'
            }
        }
    }
}
