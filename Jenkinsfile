pipeline {
  agent any

  options {
    // We do our own checkout stage
    skipDefaultCheckout(true)
  }

  environment {
    // Repository only (no tag here)
    IMAGE_REPO = 'franciswebandapp/fastapi-jenkins-exams'
    // Optional default; will be overridden per branch in deploy stage
    KUBE_NAMESPACE = ''
    DOCKER_HUB_PASS= credentials('docker-hub-credentials')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          userRemoteConfigs: [[url: 'https://github.com/Franciswp/jenkins-devops-exams.git']],
          branches: [[name: '*/main']],
          gitTool: 'git' // must match the tool name in Global Tool Configuration (or remove to use PATH)
        ])
      }
    }

    stage('Build Docker Image') {
    steps {
        withEnv(['HOME=/var/lib/jenkins']) {
        script {
            docker.build("${IMAGE_REPO}:${env.BUILD_NUMBER}")
        }
        }
    }
    }

    stage('Push to Docker Hub') {
    steps {
        withEnv(['HOME=/var/lib/jenkins']) {
        script {
            docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
            docker.image("${IMAGE_REPO}:${env.BUILD_NUMBER}").push()
            docker.image("${IMAGE_REPO}:${env.BUILD_NUMBER}").push('latest')
            }
        }
        }
    }
    }
    stage('Deploy to Kubernetes') {
      environment {
        // KUBECONFIG is a "Secret file" credential; this becomes a filepath
        KUBECONFIG = credentials('config')
      }
      when {
        not { branch 'master' } // Auto-deploy to non-prod
      }
      steps {
        script {
          // Determine namespace based on branch (defaults to 'main' if BRANCH_NAME is null)
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