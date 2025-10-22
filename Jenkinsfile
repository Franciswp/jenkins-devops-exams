pipeline {
  agent any
  options { skipDefaultCheckout(true) }
  environment {
    IMAGE_REPO = 'franciswebandapp/fastapi-jenkins-exams'
    IMAGE_TAG  = "${env.BUILD_NUMBER}"   // used by docker-compose.yaml
    KUBE_NAMESPACE = ''
  }
  stages {
      stage('Diagnostics') {
        steps {
     sh 'bash -euo pipefail -c "id; ls -l /var/run/docker.sock || true; docker version || true"'
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

    stage('Build (docker compose)') {
      steps {
        withEnv(["HOME=${env.WORKSPACE}"]) {
          sh '''
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
              set -euxo pipefail
              echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin

              # Push the tag defined in compose (IMAGE_REPO:IMAGE_TAG)
              if docker compose version >/dev/null 2>&1; then
                docker compose push app
              else
                docker-compose push app
              fi

              # Ensure 'latest' is also pushed (built via build.tags, but push it explicitly)
              docker push "${IMAGE_REPO}:latest"
            '''
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
          if (branch == 'dev')          env.KUBE_NAMESPACE = 'dev'
          else if (branch == 'qa')      env.KUBE_NAMESPACE = 'qa'
          else if (branch == 'staging') env.KUBE_NAMESPACE = 'staging'
          else                          env.KUBE_NAMESPACE = 'nonprod'

          sh """
            set -euxo pipefail
            helm upgrade --install my-app ./helm-chart \
              --namespace ${env.KUBE_NAMESPACE} \
              --set image.repository=${IMAGE_REPO} \
              --set image.tag=${IMAGE_TAG}
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
            set -euxo pipefail
            helm upgrade --install my-app ./helm-chart \
              --namespace ${env.KUBE_NAMESPACE} \
              --set image.repository=${IMAGE_REPO} \
              --set image.tag=${IMAGE_TAG}
          """
        }
      }
    }
  }
  post {
    always { cleanWs() }
  }
}