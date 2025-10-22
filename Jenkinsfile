pipeline {
  environment { // Declaration of environment variables
    DOCKER_ID = "franciswebandapp" // replace this with your docker-id
    DOCKER_IMAGE = "fastapi-jenkins-exams"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
  }
  agent any // Jenkins will be able to select all available agents
  stages {
    stage('Build (docker compose)') { // docker build image stage
      steps {
       withEnv(["HOME=${env.WORKSPACE}"])  {
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


    stage('Test Acceptance') { // we launch the curl command to validate that the container responds to the request
      steps {
        script {
          sh '''
          curl localhost
          '''
        }
      }
    }

    stage('Docker Push') { //we pass the built image to our docker hub account
      environment {
            DOCKERHUB_CREDENTIALS = credentials("docker-hub-credentials") // we retrieve docker password from secret text called docker_hub_pass saved on jenkins
        }
      steps {
        script {
          sh '''
          docker login -u $DOCKER_ID -p $DOCKERHUB_CREDENTIALS
          docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          '''
        }
      }
    }

    stage('Deployment in dev'){
      environment
      {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          ls
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          cat values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace dev
          '''
        }
      }
    }

    stage('Deployment in staging') {
      environment {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          ls
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          cat values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace staging
          '''
        }
      }
    }

    stage('Deploiement en prod'){
      environment {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
      // Create an Approval Button with a timeout of 15 minutes.
      // this requires a manual validation in order to deploy on production environment
        timeout(time: 15, unit: "MINUTES") {
            input message: 'Do you want to deploy in production ?', ok: 'Yes'
        }
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          ls
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          cat values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace prod
          '''
        }
      }
    }
  }
}