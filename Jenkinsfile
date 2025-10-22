pipeline {
    agent any

    environment {
        // Store only the repository name here (no tag)
        IMAGE_REPO = 'franciswebandapp/fastapi-jenkins-exams'
        // Optional default; will be overridden per branch in deploy stage
        KUBE_NAMESPACE = ''
    }

    stages {
        // Initial SCM checkout by Jenkins is sufficient

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_REPO}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // Use your Jenkins credentials ID for Docker Hub (Username/Password)
                    docker.withRegistry('https://index.docker.io/v1/', 'DOCKERHUB_CREDENTIALS') {
                        docker.image("${IMAGE_REPO}:${env.BUILD_NUMBER}").push()
                        docker.image("${IMAGE_REPO}:${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                not { branch 'master' } // Auto-deploy to non-prod
            }
            steps {
                script {
                    // If this is not a Multibranch Pipeline, BRANCH_NAME may be null.
                    // Adjust as needed, or set a default.
                    def branch = env.BRANCH_NAME ?: 'main'

                    if (branch == 'dev') {
                        env.KUBE_NAMESPACE = 'dev'
                    } else if (branch == 'qa') {
                        env.KUBE_NAMESPACE = 'qa'
                    } else if (branch == 'staging') {
                        env.KUBE_NAMESPACE = 'staging'
                    } else {
                        // Fallback non-prod namespace
                        env.KUBE_NAMESPACE = 'nonprod'
                    }

                    sh """
                      helm upgrade --install my-app ./helm-chart \
                        --namespace ${env.KUBE_NAMESPACE} \
                        --set image.repository=${IMAGE_REPO} \
                        --set image.tag=${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Manual Approval for Prod') {
            when { branch 'master' }
            steps {
                input message: 'Approve deployment to Prod?', ok: 'Deploy'
                script {
                    env.KUBE_NAMESPACE = 'prod'
                    sh """
                      helm upgrade --install my-app ./helm-chart \
                        --namespace ${env.KUBE_NAMESPACE} \
                        --set image.repository=${IMAGE_REPO} \
                        --set image.tag=${env.BUILD_NUMBER}
                    """
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