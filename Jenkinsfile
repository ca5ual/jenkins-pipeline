pipeline {
    agent any

    tools {
        nodejs "node" // use what I configured in tools
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

        stage ('Build docker image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'docker build -t nodemain:v1.0 .' 
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh 'docker build -t nodedev:v1.0'
                    }
                }
            }
        }

        stage ('Remove old containers') {
            steps {
                script {
                    sh 'docker rm -f $(docker ps -aq)'
                }
            }
        }

        stage ('Run containers') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'docker run -d --expose 3000 -p 3000:3000 nodemain:v1.0'
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh 'docker run -d --expose 3001 -p 3001:3000 nodedev:v1.0'
                    }
                }
            }
        }
    }
}