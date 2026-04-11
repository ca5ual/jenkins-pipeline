pipeline {
    agent any

    tools {
        nodejs "NodeJS" # use what I configured in tools
    }

    stages {
        stage ('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage ('Run tests') {
            steps {
                sh 'npm test'
            }
        }   
    }
}