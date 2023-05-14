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
    - name: helm
      image: alpine/helm:3.12.0
      command:
        - /bin/cat
      tty: true
    - name: gh
      image: maniator/gh:v2.29.0
      command:
        - /bin/cat
      tty: true
'''
        }
    }
    environment {
        DOCKER_REGISTRY = "ghcr.io"
        DOCKER_USERNAME = credentials('docker-username')
        DOCKER_PASSWORD = credentials('docker-password')
        APP_REPO = "${DOCKER_REGISTRY}/myspotontheweb/argocd-workloads-demo/pre-prod"
        APP_NAME = "demo2"
    }
    stages {

        stage('Setup') {
            steps {
                container("docker") {
                    // Configure the kubernetes builder
                    sh "docker buildx create --name k8s-builder --driver kubernetes --driver-opt replicas=1 --use"
                    // Registry login
                    sh 'echo ${DOCKER_PASSWORD} | docker login ${DOCKER_REGISTRY} --username ${DOCKER_USERNAME} --password-stdin'
                }
            }
        }

        stage('Build CI image') {
            when {
                allOf {
                    branch "main"
                    not {
                        buildingTag()
                    }
                }
                
            }
            steps {
                container("docker") {
                    sh "docker buildx build -t ${APP_REPO}/${APP_NAME}:${GIT_COMMIT} . --push"
                }
            }
        }

        stage('Build Release Candidate') {
            when {
                buildingTag()
            }
            environment {
                VERSION   = env.TAG_NAME.replaceAll('v','')
            }
            steps {
                container("docker") {
                    sh "docker buildx build -t ${APP_REPO}/${APP_NAME}:${VERSION} . --push"
                }
                container("helm") {
                    sh "echo ${DOCKER_PASSWORD} | helm registry login ${DOCKER_REGISTRY} --username ${DOCKER_USERNAME} --password-stdin"
                    sh "helm package chart --version ${VERSION} --app-version ${VERSION} --dependency-update"
                    sh "helm push ${APP_NAME}-${VERSION}.tgz oci://${APP_REPO}/charts"
                }
            }
        }

        stage('Sync with gitops repo') {
            environment {
                GITHUB_TOKEN = credentials('gitops-trigger-token')
            }
            steps {
                container("gh") {
                    sh "gh workflow run sync --repo myspotontheweb/argocd-workloads-demo"
                }

            }
        }
    }
}
