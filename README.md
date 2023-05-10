# argocd-springboot-demo2

Spring Boot demo application - Using Jenkins for CI

# Quick

## Start a cluster

```
k3d cluster create jenkins-cluster --agents=2
```

## Deploy Jenkins

```
#
# Configuration
#
cat <<END > values.yaml
controller:
  ingress:
    enabled: true
    hostName: jenkins.$(kubectl -n kube-system get service traefik -oyaml | yq .status.loadBalancer.ingress[0].ip).nip.io

  installPlugins:
    - javax-mail-api:1.6.2-9
    - kubernetes:3923.v294a_d4250b_91

  additionalPlugins:
    - job-dsl:1.83

  JCasC:
    defaultConfig: true
    configScripts:
      welcome-message: |
        jenkins:
          systemMessage: Welcome to our CI\CD server.  This Jenkins is configured and managed 'as code'.
      pipeline-job: |
        jobs:
          - script: >
              pipelineJob('argocd-springboot-demo2') {
                definition {
                  cpsScm {
                    scm {
                      git {
                        remote {
                          url('https://github.com/myspotontheweb/argocd-springboot-demo2')
                          //credentials('bitBucketUser')
                        }
                        branch('*/main')
                      }
                    }
                    scriptPath("Jenkinsfile")
                    lightweight()
                  }
                }
              }
END

#
# Deploy helm chart
#
helm upgrade jenkins jenkins --install --repo https://charts.jenkins.io -n jenkins --create-namespace -f values.yaml
```

Configure Service Account used to support Docker builds

```
kubectl create clusterrole k8s-builder --verb=get,list,create,delete --resource=deployment,pod,pods/exec
kubectl create serviceaccount k8s-builder -n jenkins
kubectl create rolebinding k8s-builder --clusterrole k8s-builder --serviceaccount=jenkins:k8s-builder -n jenkins
```

Login details:

```
cat <<END

http://$(helm -n jenkins get values jenkins | yq .controller.ingress.hostName)

$(kubectl -n jenkins get secrets jenkins -ogo-template='{{printf "user: admin\npass: %s\n" (index .data "jenkins-admin-password"|base64decode)}}')
END
```