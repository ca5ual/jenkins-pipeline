@Library('shared-lib') _

pipeline {
    agent {
        docker {
            image 'ca5ual/lab3:nodealpine-7.8.0'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
        }
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

        stage('Check Dockerfile') {
            steps {
                script {
                    sh "hadolint Dockerfile"
                }
            }
        }

        stage ('Build image') {
            steps {
                script {
                    buildImage(env.DOCKER_IMAGE, env.ENV)
                }
            }
        }

        stage('Scan image for vulnerabilities') {
            steps {
                sh """
                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,MEDIUM,LOW \
                        --no-progress \
                        ${DOCKER_IMAGE}:${ENV}-v1.0
                """
            }
        }

        stage('Push image') {
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


