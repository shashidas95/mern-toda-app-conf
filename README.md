# continuos integration

- create an ec2 instance with jenkins and docker installed, edit inbound rules for jenkins port 8080 open
- vist http://<public_url_instance>:8080
- connect the instance with ssh-agent and sudo cat /var/lib/jenkins/initialAdminPassword
- it will output password and enter the password to jenkins ui and create users and install plugins and jenkins ready
- install generic-web-hook-trigger and google-chat-notification plugin and restart the jenkins
- create personal access token of github and docker hub and url for google chat notification from github, docker hub and google-chat-notification
- use these personal access token and url for adding jenkins credentials
- In the github repo create an jenkins file after the pull request from dev to main will trigger mern-todo-app pipeline in which git clone from the github, checkout docker image build, and then push the images to dockerhub and picks the image tag from the image name which is a parameter for the second pipeline. trigger the second pipeline pass the image tag to second pipeline
- in github settings add webhook of generic-webhook-trigger with the jenkins ec2 instance in which jenkins is installed.
- create another github repo consisting of docker-compose file and k8s deployment files and in that file create an jenkins file. which receives the image tag as a parameter, git clone and checkout the k8s files to change newest received image tag and docker-compose file to change the tag of the image tag from the docker-compose file. run the docker-compose environments and sends the message to google-chat-notification app that the build is completed with the new image
- after merge the pull request from dev to main it will trigger the first pipeline and after successful build from it will trigger the second pipeline

Here's a step-by-step guide for setting up Jenkins with Docker, configuring webhooks, and triggering CI/CD pipelines in a GitHub and Docker Hub environment. This example assumes you have a basic understanding of AWS, Jenkins, Docker, Kubernetes, and GitHub.

### README.md

````markdown
# Jenkins CI/CD Pipeline for MERN Todo App

This document outlines the process for setting up a Jenkins instance on an AWS EC2 instance, configuring Jenkins with Docker and GitHub, and triggering two CI/CD pipelines for building and deploying the MERN stack application. The entire pipeline integrates GitHub, Docker Hub, and Google Chat notifications.

---

## 1. **Create EC2 Instance and Install Jenkins with Docker**

1. **Launch EC2 Instance**

   - Create an EC2 instance with an Ubuntu-based AMI .
   - Ensure that the instance has sufficient resources (e.g., t2.medium or above for Jenkins and Docker).

2. **SSH into the EC2 Instance**
   ```bash
   ssh -i your-key.pem ubuntu@<public-ip-address>
   ```
````

3. **Install Jenkins and Docker**  
   Run the following commands to install Jenkins and Docker:

   ```bash
   # Install Docker
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker

   # Install Jenkins
   sudo apt install openjdk-11-jdk
   wget -q -O - https://pkg.jenkins.io/jenkins.io.key | sudo apt-key add -
   sudo sh -c 'echo deb http://pkg.jenkins.io/debian/ / > /etc/apt/sources.list.d/jenkins.list'
   sudo apt update
   sudo apt install jenkins

   # Start Jenkins
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```

---

## 2. **Edit Inbound Rules for Jenkins Port (8080)**

1. Open the **Security Groups** settings for your EC2 instance.
2. Add an inbound rule to allow traffic on port `8080`:
   - **Type**: Custom TCP
   - **Port Range**: 8080
   - **Source**: Anywhere (0.0.0.0/0) or restrict to your IP.

---

![MERN Todo App](./screenshots/inbound%20rules.png)

## 3. **Access Jenkins UI**

1. Open your browser and visit Jenkins at:

   ```
   http://<public-ip-address>:8080
   ```

2. **Retrieve Jenkins Admin Password**  
   To get the Jenkins admin password, SSH into your EC2 instance and run:

   ```bash
   sudo cat /var/lib/jenkins/initialAdminPassword
   ```

   This will output the password for Jenkins UI.

3. **Complete Initial Setup**
   - Enter the password in the Jenkins setup page.
   - Create an admin user.
   - Install the recommended plugins.

---

## 4. **Install Required Jenkins Plugins**

1. Go to **Manage Jenkins > Manage Plugins**.
2. Install the following plugins:

   - **Generic Webhook Trigger Plugin**
   - **Google Chat Notifications Plugin**

3. **Restart Jenkins** to apply the changes.

---

## 5. **Create Personal Access Tokens**

You will need personal access tokens for **GitHub**, **Docker Hub**, and **Google Chat**.

1. **GitHub**:

   - Go to [GitHub Personal Access Tokens](https://github.com/settings/tokens).
   - Generate a token with repo and webhook permissions.

![MERN Todo App](./screenshots/pat%20github.png) 2. **Docker Hub**:

- Log in to Docker Hub.
- Navigate to **Account Settings > Security** to generate a personal access token.
  ![MERN Todo App](./screenshots/pat%20dockerhub.png)

3. **Google Chat**:
   - Create a webhook URL for Google Chat following [this guide](https://developers.google.com/chat). This will be used to send notifications.

---

![MERN Todo App](./screenshots/adding%20credentials.png)

## 6. **Configure Jenkins Credentials**

1. Go to **Manage Jenkins > Manage Credentials**.
2. Add the following credentials:
   - **GitHub Token** (Name: `github-token`)
   - **Docker Hub Token** (Name: `dockerhub-token`)
   - **Google Chat Webhook URL** (Name: `google-chat-webhook`)
     ![MERN Todo App](./screenshots/adding%20credentials.png)

---

## 7. **Create Jenkinsfile for GitHub Repository**

In your **GitHub Repository** (e.g., `mern-todo-app`), create a `Jenkinsfile` in the root directory. This file defines the pipeline stages:

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = "shashidas"
        GIT_REPO = "https://github.com/shashidas95/mern-todo-app"
        CONFIG_PROJECT_NAME = "mern-todo-app-conf"
        IMAGE_BE = "mern-todo-be"
        IMAGE_FE = "mern-todo-fe"
    }

    stages {
        stage('SETUP IMAGE TAG') {
            steps {
                script {
                    // Set the IMAGE_TAG dynamically with the current date and time
                    IMAGE_TAG = new Date().format('yyyyMMdd-HHmm')
                }
            }
        }

        stage('CLEANUP WORKSPACE') {
            steps {
                cleanWs()
            }
        }

        stage("CHECKOUT GIT REPO") {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Check Docker') {
            steps {
                sh 'echo $PATH'
                sh 'docker --version'
            }
        }

        stage("BUILD DOCKER IMAGES") {
            steps {
                script {
                    // Build backend image
                    dir('backend') {
                        sh "docker build --no-cache -t ${DOCKERHUB_USERNAME}/${IMAGE_BE}:${IMAGE_TAG} -t ${DOCKERHUB_USERNAME}/${IMAGE_BE}:latest ."
                    }

                    // Build frontend image
                    dir('frontend') {
                        sh "docker build --no-cache -t ${DOCKERHUB_USERNAME}/${IMAGE_FE}:${IMAGE_TAG} -t ${DOCKERHUB_USERNAME}/${IMAGE_FE}:latest ."
                    }
                }
            }
        }

        stage("PUSH DOCKER IMAGES TO DOCKERHUB") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh 'echo ${PASSWORD} | docker login --username ${USER_NAME} --password-stdin'

                    // Push backend image with both tags
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_BE}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_BE}:latest"

                    // Push frontend image with both tags
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_FE}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_FE}:latest"

                    sh 'docker logout'
                }
            }
        }

        stage("TRIGGERING THE CONFIG PIPELINE") {
            steps {
                // Trigger the config pipeline for backend and frontend
                build job: CONFIG_PROJECT_NAME, parameters: [string(name: 'IMAGE_TAG', value: IMAGE_TAG)]
            }
        }

        stage("CLEANUP DOCKER CACHES AND IMAGES") {
            steps {
                script {
                    // Remove all Docker images and caches
                    sh 'docker system prune -af --volumes'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully with cleanup'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}

```

---

## 8. **Configure Webhook for Generic Webhook Trigger**

In your **GitHub Repository**:

1. Go to **Settings > Webhooks**.
2. Add a new webhook with the following settings:
   - **Payload URL**: `http://<public-ip-address>:8080/generic-webhook-trigger/invoke`
   - **Content Type**: `application/json`
   - **Triggers**: Choose **Pull Request** (dev to main).

---

![MERN Todo App](./screenshots/webhook%20github.png)

## 9. **Create a Second GitHub Repository for Deployment**

Create another GitHub repository that includes Docker Compose and Kubernetes files. This repository will contain a `Jenkinsfile` for deploying the Docker image.

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', description: 'Enter the image tag')
    }

    environment {
        GIT_REPO = "https://github.com/shashidas95/mern-todo-app-conf"
        DOCKERHUB_USERNAME = "shashidas" // Docker Hub username
        IMAGE_BE = "mern-todo-be"
        IMAGE_FE = "mern-todo-fe"
        CONFIG_PROJECT_NAME = "mern-todo-app-config"
    }

    stages {
        stage('CLEANUP WORKSPACE') {
            steps {
                cleanWs()
            }
        }

        stage("CHECKOUT GIT REPO") {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage("UPDATE K8S DEPLOYMENT FILES FOR BACKEND AND FRONTEND") {
            steps {
                script {
                    // Define image names using DOCKERHUB_USERNAME and IMAGE_BE/IMAGE_FE
                    def backendImageName = "${DOCKERHUB_USERNAME}/${IMAGE_BE}"
                    def frontendImageName = "${DOCKERHUB_USERNAME}/${IMAGE_FE}"

                    // Update backend deployment file with new IMAGE_TAG
                    sh 'cat ./k8s/backend-deployment.yaml'
                    sh "sed -i 's#image: ${backendImageName}:.*#image: ${backendImageName}:${IMAGE_TAG}#' ./k8s/backend-deployment.yaml"
                    sh 'cat ./k8s/backend-deployment.yaml'

                    // Update frontend deployment file with new IMAGE_TAG
                    sh 'cat ./k8s/frontend-deployment.yaml'
                    sh "sed -i 's#image: ${frontendImageName}:.*#image: ${frontendImageName}:${IMAGE_TAG}#' ./k8s/frontend-deployment.yaml"
                    sh 'cat ./k8s/frontend-deployment.yaml'


                    // Update docker-compose  file with new IMAGE_TAG
                    sh 'cat ./docker-compose.yaml'
                    sh "sed -i 's#image: ${frontendImageName}:.*#image: ${frontendImageName}:${IMAGE_TAG}#' ./docker-compose.yaml"
                     sh "sed -i 's#image: ${backendImageName}:.*#image: ${backendImageName}:${IMAGE_TAG}#' ./docker-compose.yaml"
                    sh 'cat ./docker-compose.yaml'

                    // Configure Git for committing the changes
                    sh 'git config --global user.email "shashidas95@gmail.com"'
                    sh 'git config --global user.name "shashidas95"'
                    sh 'git add ./k8s/backend-deployment.yaml ./k8s/frontend-deployment.yaml ./docker-compose.yaml'
                    sh "git commit -m 'Updated with new ${IMAGE_TAG}'"
                    // Restarting Docker Compose
                    echo "Restarting Docker Compose"
                    sh "docker compose down"
                    sh "docker compose up -d"
                }



                // Push the changes
                withCredentials([usernamePassword(credentialsId: 'githubuser', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                    sh 'git push https://$uname:$pass@github.com/shashidas95/mern-todo-app-conf.git main'
                }
            }
        }
    }

     post {
        success {
            googlechatnotification url: 'id:google_chat_webhook', message: 'Build succeeded with new image!'
        }
        failure {
            googlechatnotification url: 'id:google_chat_webhook', message: 'Build failed!'
        }
        always {
            googlechatnotification url: 'id:google_chat_webhook', message: 'Build completed with new image tag.'
        }
    }
}

```

---

## 10. **Trigger the CI/CD Pipeline**

1. **Pull Request Merge**  
   When a pull request is merged from `dev` to `main` in the **MERN Todo App** GitHub repository, it will trigger the first Jenkins pipeline, which will:

   - Clone the repo.
   - Build the Docker image.
   - Push the image to Docker Hub.
   - Trigger the second pipeline.

2. **Second Pipeline**  
   The second pipeline:
   - Pulls the latest Docker image.
   - Updates the Docker Compose and Kubernetes files with the new image tag.
   - Deploys the app using Docker Compose.
   - Sends a notification to Google Chat.

---

## 11. **Final Workflow**

- **Step 1**: Developer pushes code to `dev` branch.
- **Step 2**: Developer creates a pull request to merge `dev` into `main`.
- **Step 3**: The first Jenkins pipeline builds and pushes the Docker image to Docker Hub.
- **Step 4**: The second Jenkins pipeline updates Docker Compose and Kubernetes, runs the Docker container, and sends a notification to Google Chat.

---

## Resources

- [Jenkins Website](https://www.jenkins.io)
- [Docker Hub](https://hub.docker.com/)
- [Google Chat Webhooks](https://developers.google.com/chat)
- [GitHub Personal Access Tokens](https://github.com/settings/tokens)

---

### Key Points:

1. **Jenkins Setup**: Installs Jenkins and Docker on an EC2 instance, configures Jenkins with necessary plugins, and sets up credentials for GitHub, Docker Hub, and Google Chat.
2. **CI/CD Pipeline**: Defines a `Jenkinsfile` for the build and deployment pipeline that involves cloning a GitHub repo, building and pushing Docker images, and triggering a second deployment pipeline.
3. **Webhooks**: Configures GitHub webhooks to trigger the Jenkins pipeline upon pull request merges.
4. **Deployment**: Automates deployment with Docker Compose and Kubernetes, and notifies Google Chat on completion.
