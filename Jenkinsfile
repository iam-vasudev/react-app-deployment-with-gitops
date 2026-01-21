pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'iamvasu/react-app'
        IMAGE_TAG = "build-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Commit Message') {
            steps {
                script {
                    def lastCommit = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
                    if (lastCommit.contains("[skip ci]") || lastCommit.contains("[ci skip]")) {
                        currentBuild.result = 'ABORTED'
                        error("Aborting build to prevent infinite loop")
                    }
                }
            }
        }

        stage('Build App') {
            steps {
                script {
                    docker.image('node:22-alpine').inside('-u root') {
                        sh 'npm install'
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """FROM nginx:alpine
COPY ./dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]"""

                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
                        sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
                        sh "docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                        sh "docker logout"
                    }
                }
            }
        }

        stage('Update Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git config user.email "vasu3a1@gmail.com"
                        git config user.name "iam-vasudev"
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        git add k8s/deployment.yaml
                        git diff --staged --quiet || git commit -m "Update image to ${IMAGE_TAG} [skip ci]"
                        git config credential.helper "!f() { echo username=\$GIT_USERNAME; echo password=\$GIT_PASSWORD; }; f"
                        git push origin HEAD:main
                    """
                }
            }
        }
    }
}