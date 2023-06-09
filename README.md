# argocd-springboot-demo2

Spring Boot demo application - Using Jenkins for CI

# Quick start

## Start a cluster

```
k3d cluster create jenkins-cluster --agents=2
```

## Deploy Jenkins

Generate configuration file

```
cat <<END > values.yaml
controller:
  ingress:
    enabled: true
    hostName: jenkins.$(kubectl -n kube-system get service traefik -oyaml | yq .status.loadBalancer.ingress[0].ip).nip.io
  additionalPlugins:
    - job-dsl:latest
    - kubernetes-credentials-provider:latest
  JCasC:
    defaultConfig: true
    configScripts:
      pipeline-job: |
        jobs:
          - script: >
              multibranchPipelineJob('argocd-springboot-demo2') {
                branchSources {
                  branchSource {
                    source {
                      git {
                        id('argocd-springboot-demo2')
                        remote('https://github.com/myspotontheweb/argocd-springboot-demo2.git')
                        traits {
                          gitBranchDiscovery()
                          gitTagDiscovery()
                          cloneOption {
                            extension {
                              noTags(false)
                              shallow(true)
                              reference("")
                              timeout(2)
                            }
                          }
                        }
                      }
                    }
                  }
                }
                orphanedItemStrategy {
                  discardOldItems {
                    numToKeep(20)
                  }
                }
              }
END
```

Install helm chart

```
helm upgrade jenkins jenkins --install --repo https://charts.jenkins.io -n jenkins --create-namespace -f values.yaml
```

Configure Service Accounts

```
#
# Service account "jenkins:k8s-builder" (used by k8s agent pods)
#
kubectl create clusterrole k8s-builder --verb=get,list,create,delete --resource=deployment,pod,pods/exec
kubectl create serviceaccount k8s-builder -n jenkins
kubectl create rolebinding k8s-builder --clusterrole k8s-builder --serviceaccount=jenkins:k8s-builder -n jenkins

#
# Service account "jenkins:jenkins" (used by Jenkins pod)
#
# Fix for https://issues.jenkins.io/browse/JENKINS-62646
#
kubectl create clusterrolebinding jenkinsrolebinding --clusterrole=cluster-admin --serviceaccount=jenkins:jenkins
```

Configure [Build secrets](https://github.com/jenkinsci/kubernetes-credentials-provider-plugin)

```
USER=XXXXXXXXXXXXXXXX
PASS=YYYYYYYYYYYYYYYY
TOKEN=ZZZZZZZZZZZZZZZ
```

Generate secrets
```
kubectl -n jenkins create secret generic docker-username --from-literal text=$USER
kubectl -n jenkins label secret docker-username jenkins.io/credentials-type=secretText
kubectl -n jenkins annotate secret docker-username jenkins.io/credentials-description="Username to access docker registry"

kubectl -n jenkins create secret generic docker-password --from-literal text=$PASS
kubectl -n jenkins label secret docker-password jenkins.io/credentials-type=secretText
kubectl -n jenkins annotate secret docker-password jenkins.io/credentials-description="Password to access docker registry"

kubectl -n jenkins create secret generic gitops-trigger-token --from-literal text=$TOKEN
kubectl -n jenkins label secret gitops-trigger-token jenkins.io/credentials-type=secretText
kubectl -n jenkins annotate secret gitops-trigger-token jenkins.io/credentials-description="Token used to trigger GH action"
```

## Login

```
cat <<END

http://$(helm -n jenkins get values jenkins | yq .controller.ingress.hostName)

$(kubectl -n jenkins get secrets jenkins -ogo-template='{{printf "user: admin\npass: %s\n" (index .data "jenkins-admin-password"|base64decode)}}')
END
```

