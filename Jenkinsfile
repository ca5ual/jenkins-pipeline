@Library('shared-lib') _

pipeline {
    agent {
        docker {
            image 'ca5ual/lab3:nodealpine-7.8.0'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
        }
    }

    environment {
        DOCKER_REPO = "ca5ual/lab3"
        V_TAG = "v1.0"
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
                        env.TAG = 'nodemain'
                        env.JOB = 'Deploy_to_main'
                    } else if (env.BRANCH_NAME == 'dev') {
                        env.TAG = 'nodedev'
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
                    buildImage(env.DOCKER_REPO, env.TAG)
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
                        ${DOCKER_REPO}:${TAG}-${V_TAG}
                """
            }
        }

        stage('Push image') {
            steps {
                script {
                    pushImage(env.DOCKER_REPO, env.TAG, "docker-creds")
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


