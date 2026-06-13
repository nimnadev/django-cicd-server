pipeline {
  agent any
  environment {
    DOCKERHUB_USER = 'YOUR_DOCKERHUB_USERNAME'
    IMAGE_NAME     = 'YOUR_DOCKERHUB_USERNAME/django-app'
    GITHUB_USER    = 'YOUR_GITHUB_USERNAME'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
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
        sh 'docker build -t \:\ .'
      }
    }
    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DUSER',
          passwordVariable: 'DPASS'
        )]) {
          sh 'echo \ | docker login -u \ --password-stdin'
          sh 'docker push \:\'
          sh 'docker tag \:\ \:latest'
          sh 'docker push \:latest'
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
          sh 'rm -rf django-k8s'
          sh 'git clone https://\:\@github.com/\/django-k8s.git'
          sh 'sed -i "s|django-app:.*|django-app:\|g" django-k8s/deployment.yaml'
          sh 'cd django-k8s && git config user.email "jenkins@ci.com"'
          sh 'cd django-k8s && git config user.name "Jenkins"'
          sh 'cd django-k8s && git add deployment.yaml'
          sh 'cd django-k8s && git commit -m "Update image to \"'
          sh 'cd django-k8s && git push origin main'
        }
      }
    }
  }
  post {
    success { echo 'Done! ArgoCD will deploy shortly.' }
    failure { echo 'Pipeline failed. Check stage logs.' }
  }
}
