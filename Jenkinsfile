pipeline {
  agent any

  environment {
    DOCKERHUB_USER = 'nimnajudy'
    IMAGE_NAME     = 'nimnajudy/django-app'
    GITHUB_USER    = 'nimnadev'
    BUILD_TAG      = "v${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'sonar-scanner'
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${BUILD_TAG} ."
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DUSER',
          passwordVariable: 'DPASS'
        )]) {
          sh "echo ${DPASS} | docker login -u ${DUSER} --password-stdin"
          sh "docker push ${IMAGE_NAME}:${BUILD_TAG}"
          sh "docker tag ${IMAGE_NAME}:${BUILD_TAG} ${IMAGE_NAME}:latest"
          sh "docker push ${IMAGE_NAME}:latest"
        }
      }
    }

    stage('Update K8s Manifest') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-creds',
          usernameVariable: 'GUSER',
          passwordVariable: 'GPASS'
        )]) {
          sh """
            rm -rf django-k8s
            git clone https://${GUSER}:${GPASS}@github.com/nimnadev/django-k8s.git
            sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${BUILD_TAG}|g" django-k8s/deployment.yaml
            cd django-k8s
            git config user.email "jenkins@ci.com"
            git config user.name "Jenkins"
            git add deployment.yaml
            git commit -m "Update image to ${BUILD_TAG}"
            git push origin main
          """
        }
      }
    }
  }

  post {
    success {
      echo 'Done! ArgoCD will deploy shortly.'
    }
    failure {
      echo 'Pipeline failed. Check stage logs.'
    }
  }
}