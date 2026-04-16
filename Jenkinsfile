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

        stage ('Set ENV') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.ENV = 'nodemain'
                        env.PORT = '3000'
                        env.TAG = "v1.0"
                    } else if (env.BRANCH_NAME == 'dev') {
                        env.ENV = 'nodedev'
                        env.PORT = '3001'
                        env.TAG = "v1.0"
                    }
                }
            }
        }

        stage ('Build docker image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'docker build -t ${ENV}:v1.0 .' 
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh 'docker build -t ${ENV}:v1.0 .'
                    }
                }
            }
        }

        stage ('Remove old containers') {
            steps {
                script {
                    sh 'docker rm -f ${ENV} || true'
                }
            }
        }

        stage ('Run containers') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'docker run -d --name ${ENV} --expose ${PORT} -p ${PORT}:${PORT} ${ENV}:${TAG}'
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh 'docker run -d --name ${ENV} --expose ${PORT} -p ${PORT}:${PORT} ${ENV}:${TAG}'
                    }
                }
            }
        }
    }
}