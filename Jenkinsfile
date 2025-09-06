pipeline {
  agent { label 'agent-1' } // Run on your Jenkins agent node
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  parameters {
    string(name: 'DOCKERHUB_REPO', defaultValue: 'yourdockerhubuser/java-eks-demo', description: 'DockerHub repo, e.g., username/reponame')
    string(name: 'EKS_CLUSTER', defaultValue: 'your-eks-cluster', description: 'EKS cluster name')
    string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region for the cluster')
    string(name: 'K8S_NAMESPACE', defaultValue: 'demo', description: 'Kubernetes namespace to deploy into')
  }
  environment {
    DOCKERHUB_CREDS = 'dockerhub-creds' // Create this in Jenkins Credentials
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git --version && mvn -version && docker --version'
      }
    }
    stage('Build & Unit Tests') {
      steps {
        sh 'mvn -B -DskipTests=false test'
      }
    }
    stage('Package') {
      steps {
        sh 'mvn -B -DskipTests package'
      }
    }
    stage('Set Image Tag') {
      steps {
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.IMAGE = "${params.DOCKERHUB_REPO}:${env.GIT_COMMIT_SHORT}"
        }
        echo "Using image: ${env.IMAGE}"
      }
    }
    stage('Docker Build & Push') {
      steps {
        script {
          sh "docker build -t ${env.IMAGE} -t ${params.DOCKERHUB_REPO}:latest ."
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDS, usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
            sh 'echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin'
            sh "docker push ${env.IMAGE}"
            sh "docker push ${params.DOCKERHUB_REPO}:latest"
            sh 'docker logout'
          }
        }
      }
    }
    stage('Update K8s Manifests') {
      steps {
        sh """sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}|g" k8s/deployment.yaml"""
        sh 'echo "Updated image:" && grep -n "image:" k8s/deployment.yaml'
      }
    }
    stage('Configure kubeconfig') {
      steps {
        sh "aws eks update-kubeconfig --name ${params.EKS_CLUSTER} --region ${params.AWS_REGION}"
        sh 'kubectl version --client --output=yaml || true'
      }
    }
    stage('Deploy to EKS') {
      steps {
        sh "kubectl get ns ${params.K8S_NAMESPACE} || kubectl create ns ${params.K8S_NAMESPACE}"
        sh "kubectl apply -n ${params.K8S_NAMESPACE} -f k8s/"
        sh "kubectl rollout status deployment/demo-deployment -n ${params.K8S_NAMESPACE} --timeout=180s"
        sh "echo 'Service hostname:' && kubectl get svc -n ${params.K8S_NAMESPACE} demo-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}' || true"
      }
    }
  }
  post {
    always {
      junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false
      cleanWs()
    }
  }
}
