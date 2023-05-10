pipeline {
    agent {
        kubernetes {yaml '''
apiVersion: v1
kind: Pod
metadata:
labels:
  project-name: demo
spec:
  serviceAccountName: k8s-builder
  containers:
    - name: docker
      image: docker:23.0.4-cli
      tty: true
'''
        }
    }
    environment {
        REG_HOST = "ghcr.io"
        REG_USER = credentials('docker-username')
        REG_PASS = credentials('docker-password')
        IMAGE_REPO = 'myspotontheweb/argocd-workloads-demo/pre-prod/demo2'
        IMAGE_TAG  = '1.0.0'
    }
    stages {
        stage('Setup') {
            steps {
                container("docker") {
                    // Configure the kubernetes builder
                    sh "docker buildx create --name k8s-builder --driver kubernetes --driver-opt replicas=1 --use"
                    // Registry login
                    sh 'echo $REG_PASS | docker login ${REG_HOST} --username ${REG_USER} --password-stdin'
                }
            }
        }
        stage('Build') {
            steps {
                container("docker") {
                    // git checkout
                    git url: 'https://github.com/myspotontheweb/argocd-springboot-demo2.git', branch: 'main'
                    // Build + Push image
                    sh "docker buildx build -t ${REG_HOST}/${IMAGE_REPO}:${IMAGE_TAG} . --push"
                }
            }
        }
    }
}
