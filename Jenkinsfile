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
