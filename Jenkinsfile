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
                withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh '''
                        git config user.email "vasu3a1@gmail.com"
                        git config user.name "iam-vasudev"
                        
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" k8s/deployment.yaml
                        
                        git add k8s/deployment.yaml
                        git commit -m "Update deployment image to ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        
                        # Configure git to use the environment variables for authentication
                        # This avoids putting the username/password in the URL
                        git config credential.helper "!f() { echo username=\$GIT_USERNAME; echo password=\$GIT_PASSWORD; }; f"
                        
                        # Just push to origin (credentials will be provided by the helper)
                        git push origin HEAD:main
                    '''
                }
            }
        }
    }
}
