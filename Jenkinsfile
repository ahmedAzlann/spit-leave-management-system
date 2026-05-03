pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Nitin15122005/spit-leave-management-system.git'
            }
        }

        stage('Stop Existing Containers') {
            steps {
                sh 'docker compose down || true'
            }
        }

        stage('Build & Deploy') {
            steps {
                sh 'docker compose up --build -d'
            }
        }
    }
}
