# Jenkins Pipeline 

A complete **CI/CD pipeline** for a Node.js application that automatically builds, tests, scans, and deploys code to different environments based on the Git branch.

---

## What Does This Pipeline Do?


1. **Install**
2. **Test** 
3. **Check**
4. **Build** 
5. **Scan**
6. **Push** 
7. **Trigger**

Everything happens automatically whenever you push code to GitHub

---

## Pipeline Architecture

### Agent Configuration
```groovy
agent {
    docker {
        image 'ca5ual/lab3:nodealpine-7.8.0'
        args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
    }
}
```

**What this means:**
- The pipeline runs **inside a Docker container** - >Mounts Docker socket to allow **Docker-in-Docker** - > Runs as root to access Docker

### Environment Variables
```groovy
DOCKER_REPO = "ca5ual/lab3"      # Where images are pushed
V_TAG = "v1.0"                    # Version tag for images
```

---

## Pipeline Stages Explained

### Install Dependencies
```groovy
sh 'npm install'
```
Downloads and installs all Node.js packages listed in `package.json`

---

### Run Tests
```groovy
sh 'npm test'
```
Executes the test suite to verify the application works correctly.  
---

### Set ENV
```groovy
if (env.BRANCH_NAME == 'main') {
    env.TAG = 'nodemain'
    env.JOB = 'Deploy_to_main'
} else if (env.BRANCH_NAME == 'dev') {
    env.TAG = 'nodedev'
    env.JOB = 'Deploy_to_dev'
}
```

**Branch detection:**
- **main branch** → tag: `nodemain` → triggers `Deploy_to_main` job (production)
- **dev branch** → tag: `nodedev` → triggers `Deploy_to_dev` job (staging)

---

### Check Dockerfile
```groovy
sh "hadolint Dockerfile"
```
Ensures Dockerfiles follow best practices before building

---

### Build Image
```groovy
buildImage(env.DOCKER_REPO, env.TAG)
```

**Custom function from shared library** that:
- Builds a Docker image from the Dockerfile
- Tags it as: `ca5ual/lab3:nodemain-v1.0` (or `nodedev-v1.0`)
- Creates a reproducible, containerized version of the app

---

### Scan Image for Vulnerabilities
```groovy
trivy image \
    --exit-code 0 \
    --severity HIGH,MEDIUM,LOW \
    --no-progress \
    ${DOCKER_REPO}:${TAG}-${V_TAG}
```

Uses **Trivy** to scan the Docker image for known CVEs:
- Checks for vulnerabilities in OS packages
- Checks for vulnerabilities in app dependencies
- Reports **HIGH, MEDIUM, and LOW** severity issues
- `--exit-code 0` = scan completes even if vulnerabilities found (warnings only, won't fail the pipeline)

---

### Push Image
```groovy
pushImage(env.DOCKER_REPO, env.TAG, "docker-creds")
```

**Custom function from shared library** that:
- Authenticates to Docker Hub using stored credentials (`docker-creds`)
- Pushes the built image to the registry: `ca5ual/lab3:nodemain-v1.0`
- Makes the image available for deployment

---

### Trigger Deploy
```groovy
build job: env.JOB
```

**Triggers a downstream Jenkins job:**
- If main branch → runs `Deploy_to_main` (production deployment)
- If dev branch → runs `Deploy_to_dev` (staging deployment)

This separates CI (build & test) from CD (deploy) into manageable jobs

---

##  Deployment Jobs: Deploy_to_dev & Deploy_to_main

After the CI pipeline completes successfully, one of these deployment jobs is triggered automatically.

### Deploy_to_dev Job

```groovy
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
        ENV = "nodedev"
        PORT = "3001"
    }
    stages {
        stage('Pull image') {
            steps {
                script {
                    pullImage(env.DOCKER_REPO, env.ENV)
                }
            }
        }
        stage('Remove old container') {
            steps {
                script {
                    removeContainer(env.ENV)
                }
            }
        }
        stage ('Deploy container') {
            steps {
                script {
                    deployContainer(env.ENV, env.DOCKER_REPO, env.PORT)
                }
            }
        }
    }
}
```

**Stages:**
1. **Pull image** → downloads `ca5ual/lab3:nodedev-v1.0` from Docker Hub
2. **Remove old container** → stops and removes the old `nodedev` container (if exists)
3. **Deploy container** → starts a new container on port 3001

**Result:** Your dev version is live on `http://server:3001`

---

### Deploy_to_main Job

```groovy
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
        ENV = "nodemain"
        PORT = "3000"
    }
    stages {
        stage('Pull image') {
            steps {
                script {
                    pullImage(env.DOCKER_REPO, env.ENV)
                }
            }
        }
        stage('Remove old container') {
            steps {
                script {
                    removeContainer(env.ENV)
                }
            }
        }
        stage ('Deploy container') {
            steps {
                script {
                    deployContainer(env.ENV, env.DOCKER_REPO, env.PORT)
                }
            }
        }
    }
}
```
**Stages:** Same 3-stage process as dev, but with production configuration

**Result:** Your production version is live on `http://server:3000`

---

## Shared Library Integration

This pipeline uses a **shared Jenkins library** (`@Library('shared-lib')`) for reusable functions:

### Shared Library Functions Explained

#### `buildImage(repo, env)`
**File:** `vars/buildimage.groovy`

```groovy
def call(String repo, String env) {
    sh "docker build -t ${repo}:${env}-v1.0 ."
}
```
- Takes the Dockerfile in the current directory
- Creates a Docker image with the tag `ca5ual/lab3:nodemain-v1.0` or `ca5ual/lab3:nodedev-v1.0`

---

#### `pushImage(repo, env, credentialsId)`

```groovy
def call(String repo, String env, String credentialsId) {
    withCredentials([usernamePassword(
        credentialsId: credentialsId,
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )]) {
        sh """
        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
        docker push ${repo}:${env}-v1.0
        """
    }
}
```

**What it does:**
1. **Securely retrieves** Docker Hub credentials from Jenkins (stored as `docker-creds`)
2. **Logs in** to Docker Hub using username & password
3. **Pushes** the image to Docker Hub registry

---

#### `pullImage(repo, env)`

```groovy
def call(String repo, String env) {
    sh "docker pull ${repo}:${env}-v1.0"
}
```
- Downloads the Docker image from Docker Hub to the local machine/server

---

#### `deployContainer(env, repo, port)`

```groovy
def call(String env, String repo, String port) {
    sh "docker run -d --name ${env} -p ${port}:3000 ${repo}:${env}-v1.0"
}
```

**Not called in THIS pipeline**, but used in 'Deploy_to_main' and 'Deploy_to_dev' jobs

---

#### `removeContainer(containerName)`

```groovy
def call(String containerName) {
    sh "docker rm -f ${containerName} || true"
}
```
- Clean up old containers before deploying new ones
---

### Function Call Flow in This Pipeline

```
pipeline starts
    ↓
[CI stages: install, test, build, scan, push]
    ↓
buildImage("ca5ual/lab3", "nodemain")  ← from shared library
    ↓
pushImage("ca5ual/lab3", "nodemain", "docker-creds")  ← from shared library
    ↓
trigger downstream job: "Deploy_to_main"
    ↓
[Deployment job uses pullImage & deployContainer & removeContainer]
```

---

## How to Use This Pipeline

### Prerequisites
1. **Jenkins** running with Docker support
2. **GitHub** repository with this Jenkinsfile
3. **Docker Hub** account and credentials stored in Jenkins as `docker-creds`
4. **Shared library** configured in Jenkins (pointing to the shared-lib repo)

---

## Security Features

1. **Dockerfile Linting** (Hadolint) - catches misconfigurations
2. **Image Scanning** (Trivy) - finds CVEs before deployment
3. **Credentials Management** - Docker credentials stored securely in Jenkins
4. **Branch Separation** - main and dev pipelines are isolated

---

## Key Technologies

| Tool | Purpose |
|------|---------|
| **Jenkins** | CI/CD orchestration |
| **Docker** | Containerization & execution environment |
| **Node.js** | JavaScript runtime for the app |
| **npm** | Dependency management & testing |
| **Hadolint** | Dockerfile linting |
| **Trivy** | Container security scanning |
| **Docker Hub** | Image registry |
