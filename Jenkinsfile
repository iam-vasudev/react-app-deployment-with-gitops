pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'iamvasu/react-app' // Replace with your DockerHub info
        KubeConfig = 'k8s-secret' // Replace with your Jenkins credential ID for Kubeconfig
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                    // Create Dockerfile on the fly
                    writeFile file: 'Dockerfile', text: """FROM nginx:alpine
COPY ./dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]"""

                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') { // Renaming to a common default 'docker-credentials' in case that's what user used.
                        def appImage = docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                        appImage.push()
                        appImage.push('latest')
                    }
                }
            }
        }

        stage('Update Manifest') {
            steps {
                withCredentials([string(credentialsId: 'git-credentials', variable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" k8s/deployment.yaml
                        
                        git add k8s/deployment.yaml
                        git commit -m "Update deployment image to ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        
                        # Assuming remote name is origin. Adjust if necessary.
                        # We need to set the URL with the token for pushing.
                        # Using the token directly in the URL: https://<token>@github.com/...
                        git remote set-url origin https://${GIT_TOKEN}@github.com/iam-vasudev/react-app-deployment-with-gitops.git
                        
                        git push origin HEAD:main
                    '''
                }
            }
        }
    }
}
