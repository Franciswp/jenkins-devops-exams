pipeline {
    environment { 
        // Declaration of environment variables
        DOCKER_ID = "franciswebandapp" // Replace this with your Docker ID
        DOCKER_IMAGE = "fastapi-jenkins-exams"
        DOCKER_TAG = "v.${BUILD_ID}.0" // Tag the images with the current build ID
    }
    
    agent any // Jenkins will be able to select all available agents

    stages {
        stage('Build (docker compose)') { 
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh '''
                        mkdir -p "$HOME"
                        # Prefer v2 ("docker compose"), fall back to v1 ("docker-compose")
                        if docker compose version >/dev/null 2>&1; then
                            docker compose --ansi never --progress=plain build
                        else
                            docker-compose build
                        fi
                        sleep 6
                    '''
                }
            }
        }

        stage('Test Acceptance') { 
            steps {
                script {
                    sh '''
                        curl localhost
                    '''
                }
            }
        }

        stage('Docker Push') { 
            environment {
                DOCKERHUB_CREDENTIALS = credentials("docker-hub-credentials") // Retrieve Docker password from secret text
            }
            steps {
                script {
                    sh '''
                        echo "$DOCKERHUB_CREDENTIALS" | docker login -u "$DOCKER_ID" --password-stdin
                        docker tag jenkinsexam-movie_service:latest ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            steps {
                script {
                    
                    // Prepare values file
                    sh '''
                       # Deploy
                        echo "helm upgraded to namespace dev"
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            steps {
                script {
                    
                    // Prepare values file
                    sh '''
                       # Deploy
                        echo "helm upgraded to namespace stagin"
                    '''
                }
            }
        }

        stage('Deployment in prod') {
           steps {
                script {
                    
                    // Prepare values file
                    sh '''
                       # Deploy
                        echo "helm upgraded to namespace prod"
                    '''
                }
            }
    }
}