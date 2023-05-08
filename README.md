# argocd-springboot-demo2

Spring Boot demo application - Using Jenkins for CI

# Quick

Start a cluster

```
k3d cluster create jenkins-cluster --agents=2
```

Deploy Jenkins

```
#
# Configuration
#
cat <<END > values.yaml
controller:
   ingress:
       enabled: true
       paths: []
       apiVersion: "extensions/v1beta1"
       hostName: jenkins.$(kubectl -n kube-system get service traefik -oyaml | yq .status.loadBalancer.ingress[0].ip).nip.io
END

#
# Deploy helm chart
#
helm upgrade jenkins jenkins --install --repo https://charts.jenkins.io -n jenkins --create-namespace -f values.yaml
```

Login details:

```
cat <<END

http://$(helm -n jenkins get values jenkins | yq .controller.ingress.hostName)

$(kubectl -n jenkins get secrets jenkins -ogo-template='{{printf "user: admin\npass: %s\n" (index .data "jenkins-admin-password"|base64decode)}}')
END
```