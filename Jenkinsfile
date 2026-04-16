@Library('shared-lib') _

pipeline {
    agent any

    tools {
        nodejs "node" // use what I configured in tools
    }

    environment {
        DOCKER_IMAGE = "ca5ual/lab3"
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
                        env.JOB = 'Deploy_to_main'
                    } else if (env.BRANCH_NAME == 'dev') {
                        env.ENV = 'nodedev'
                        env.JOB = 'Deploy_to_dev'
                    }
                }
            }
        }

        stage ('Build docker image') {
            steps {
                script {
                    buildImage(env.DOCKER_IMAGE, env.ENV)
                }
            }
        }

        stage('Push to docker') {
            steps {
                script {
                    pushImage(env.DOCKER_IMAGE, env.ENV, "docker-creds")
                    }
                }
            }

        stage ('Trigger deploy') {
            steps {
                script {
                    build job: env.JOB
                }
            }
        }
    }
}

