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
    - name: gh
      image: maniator/gh:v2.29.0
      tty: true
'''
        }
    }
    environment {
        REGISTRY = "ghcr.io"
        REG_USER = credentials('docker-username')
        REG_PASS = credentials('docker-password')
        APP_REPO = "${REGISTRY}/myspotontheweb/argocd-workloads-demo/pre-prod"
        APP_NAME = demo2
    }
    stages {

        stage('Setup') {
            steps {
                container("docker") {
                    // Configure the kubernetes builder
                    sh "docker buildx create --name k8s-builder --driver kubernetes --driver-opt replicas=1 --use"
                    // Registry login
                    sh 'echo $REG_PASS | docker login ${REGISTRY} --username ${REG_USER} --password-stdin'
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
            environment {
                IMAGE_TAG = "${GIT_COMMIT}"
            }
            steps {
                container("docker") {
                    sh "docker buildx build -t ${APP_REPO}/${APP_NAME}:${IMAGE_TAG} . --push"
                }
            }
        }

        stage('Build Release Candidate') {
            when {
                buildingTag()
            }
            environment {
                IMAGE_TAG = "${TAG_NAME}"
                VERSION   = "$(TAG_NAME#v)"
            }
            steps {
                container("docker") {
                    sh "docker buildx build -t ${APP_REPO}/${APP_NAME}:${IMAGE_TAG} . --push"
                }
                container("helm") {
                    sh "helm package chart --version ${VERSION} --app-version ${IMAGE_TAG} --dependency-update"
                    sh "helm push ${env.APP_NAME}-${VERSION}.tgz oci://${APP_REPO}/charts"
                }
            }
        }

        stage('Sync with gitops repo') {
            environment {
                GITHUB_TOKEN: credentials('gitops-trigger-token')
            }
            steps {
                container("gh") {
                    sh "gh workflow run sync --repo myspotontheweb/argocd-workloads-demo"
                }

            }
        }
    }
}
