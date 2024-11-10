pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', description: 'Enter the image tag')
    }
    
    environment {
        GIT_REPO = "https://github.com/shashidas95/todo" 
        DOCKERHUB_USERNAME = "shashidas" // Docker Hub username
        IMAGE_BE = "todo-be" // Define backend image name
        IMAGE_FE = "todo-fe" // Define frontend image name
        CONFIG_PROJECT_NAME = "todo-config"
    }

    stages {
        stage('CLEANUP WORKSPACE') {
            steps {
                cleanWs()
            }
        }

        stage("CHECKOUT GIT REPO") {
            steps {
                git branch: 'phase1', url: "${GIT_REPO}"
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

                    // Configure Git for committing the changes
                    sh 'git config --global user.email "shashidas95@gmail.com"'
                    sh 'git config --global user.name "shashidas95"'
                    sh 'git add ./k8s/backend-deployment.yaml ./k8s/frontend-deployment.yaml'
                    sh "git commit -m 'Updated backend and frontend deployment files to ${IMAGE_TAG}'"
                }

                // Push the changes
                withCredentials([usernamePassword(credentialsId: 'githubuser', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                    sh 'git push https://$uname:$pass@github.com/shashidas95/todo.git phase1'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
