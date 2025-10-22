pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    IMAGE_REPO = 'franciswebandapp/fastapi-jenkins-exams'
    KUBE_NAMESPACE = ''
  }

  stages {
    stage('Diagnostics') {
      steps {
        sh '''
          echo "User and groups:"
          id
          echo "Docker socket permissions:"
          ls -l /var/run/docker.sock || true
          echo "Docker version:"
          docker version || true
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          userRemoteConfigs: [[url: 'https://github.com/Franciswp/jenkins-devops-exams.git']],
          branches: [[name: '*/main']],
          gitTool: 'git'
        ])
      }
    }

    stage('Build Docker Image') {
      steps {
        // If /home/jenkins isn’t writable, use workspace as HOME to avoid permission issues
        withEnv(["HOME=${env.WORKSPACE}"]) {
          sh 'mkdir -p "$HOME"'
          script {
            docker.build("${IMAGE_REPO}:${env.BUILD_NUMBER}")
          }
        }
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withEnv(["HOME=${env.WORKSPACE}"]) {
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
        KUBECONFIG = credentials('config')
      }
      when {
        not { branch 'master' }
      }
      steps {
        script {
          def branch = env.BRANCH_NAME ?: 'main'
          if (branch == 'dev')      env.KUBE_NAMESPACE = 'dev'
          else if (branch == 'qa')  env.KUBE_NAMESPACE = 'qa'
          else if (branch == 'staging') env.KUBE_NAMESPACE = 'staging'
          else                      env.KUBE_NAMESPACE = 'nonprod'

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
    always { cleanWs() }
  }
}