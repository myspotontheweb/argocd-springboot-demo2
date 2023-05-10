# argocd-springboot-demo2

Spring Boot demo application - Using Jenkins for CI

# Quick

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
    - job-dsl:1.83
    - kubernetes-credentials-provider:1.211.vc236a_f5a_2f3c

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
```

Install helm chart

```
helm upgrade jenkins jenkins --install --repo https://charts.jenkins.io -n jenkins --create-namespace -f values.yaml
```

This is a fix for [JENKINS-62646](https://issues.jenkins.io/browse/JENKINS-62646)

```
#
# TODO: Cluster admin is too much access to grant
#
kubectl create clusterrolebinding jenkinsrolebinding --clusterrole=cluster-admin --serviceaccount=jenkins:jenkins
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

Build secrets

```
USER=XXXXXXXXXXXXXXXX
PASS=YYYYYYYYYYYYYYYY

kubectl -n jenkins create secret generic docker-username --from-literal text=$USER
kubectl -n jenkins label secret docker-username jenkins.io/credentials-type=secretText
kubectl -n jenkins annotate secret docker-username jenkins.io/credentials-description="Username to access docker registry"

kubectl -n jenkins create secret generic docker-password --from-literal text=$PASS
kubectl -n jenkins label secret docker-password jenkins.io/credentials-type=secretText
kubectl -n jenkins annotate secret docker-password jenkins.io/credentials-description="Password to access docker registry"
```