pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_REPO     = 'franciswebandapp/fastapi-jenkins-exams'
        IMAGE_TAG      = env.BUILD_NUMBER // Used by docker-compose.yaml
        KUBE_NAMESPACE = ''
    }

    stages {
        stage('Checkout') {
            steps {
                // Modern, cleaner syntax for checking out from Git
                git url: 'https://github.com/Franciswp/jenkins-devops-exams.git', branch: 'main'
            }
        }

        stage('Build (docker compose)') {
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh '''
                        #!/usr/bin/env bash
                        set -euxo pipefail
                        mkdir -p "$HOME"
                        # Prefer v2 ("docker compose"), fall back to v1 ("docker-compose")
                        if docker compose version >/dev/null 2>&1; then
                            docker compose --ansi never --progress=plain build
                        else
                            docker-compose build
                        fi
                    '''
                }
            }
        }

        stage('Push (docker compose)') {
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                        sh '''
                            #!/usr/bin/env bash
                            set -euxo pipefail
                            echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                            # Push the tag defined in compose (IMAGE_REPO:IMAGE_TAG)
                            if docker compose version >/dev/null 2>&1; then
                                docker compose push app
                            else
                                docker-compose push app
                            fi
                            # Ensure 'latest' is also pushed
                            docker push "${IMAGE_REPO}:latest"
                        '''
                    }
                }
            }
        }

        stage('Deploy to Non-Production') {
            environment {
                // Load Kubeconfig file from Jenkins credentials
                KUBECONFIG = credentials('config')
            }

            // This stage runs for any branch that is NOT 'main'
            when {
                not { branch 'main' }
            }

            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'main'
                    if (branch == 'dev')          env.KUBE_NAMESPACE = 'dev'
                    else if (branch == 'qa')      env.KUBE_NAMESPACE = 'qa'
                    else if (branch == 'staging') env.KUBE_NAMESPACE = 'staging'
                    else                          env.KUBE_NAMESPACE = 'nonprod' // Default for other branches

                    sh '''
                        #!/usr/bin/env bash
                        set -euxo pipefail
                        helm upgrade --install my-app ./helm-chart \
                            --namespace "${env.KUBE_NAMESPACE}" \
                            --set image.repository="${IMAGE_REPO}" \
                            --set image.tag="${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Manual Approval for Production') {
            // This stage only runs for the 'main' branch
            when {
                branch 'main'
            }
            steps {
                input message: 'Approve deployment to Production?', ok: 'Deploy'
            }
        }

        stage('Deploy to Production') {
            environment {
                KUBECONFIG = credentials('config')
            }

            // This stage only runs for the 'main' branch after manual approval
            when {
                branch 'main'
            }

            steps {
                script {
                    env.KUBE_NAMESPACE = 'prod'
                    sh '''
                        #!/usr/bin/env bash
                        set -euxo pipefail
                        helm upgrade --install my-app ./helm-chart \
                            --namespace "${env.KUBE_NAMESPACE}" \
                            --set image.repository="${IMAGE_REPO}" \
                            --set image.tag="${IMAGE_TAG}"
                    '''
                }
            }
        }
    }

    post {
        always {
            // Clean up the workspace after the pipeline run
            cleanWs()
        }
    }
}