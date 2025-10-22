pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('DOCKERHUB_CREDENTIALS')  // Add this in Jenkins credentials
        IMAGE_NAME = 'franciswebandapp/fastapi-jenkins-exams'
        KUBE_NAMESPACE = ''  // Will set based on branch
    }
    
    stages {
        // Removed custom 'Checkout' stage - initial SCM checkout is sufficient
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS) {
                        docker.image("${IMAGE_NAME}:${env.BUILD_NUMBER}").push()
                        docker.image("${IMAGE_NAME}:${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                not { branch 'master' }  // Auto-deploy to non-prod
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'dev') { KUBE_NAMESPACE = 'dev' }
                    else if (env.BRANCH_NAME == 'qa') { KUBE_NAMESPACE = 'qa' }
                    else if (env.BRANCH_NAME == 'staging') { KUBE_NAMESPACE = 'staging' }
                    
                    sh "helm upgrade --install my-app ./helm-chart --namespace ${KUBE_NAMESPACE} --set image.repository=${IMAGE_NAME} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Manual Approval for Prod') {
            when { branch 'master' }
            steps {
                input message: 'Approve deployment to Prod?', ok: 'Deploy'
                script {
                    KUBE_NAMESPACE = 'prod'
                    sh "helm upgrade --install my-app ./helm-chart --namespace prod --set image.repository=${IMAGE_NAME} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}