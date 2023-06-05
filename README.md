## Install Jenkins server
### Set up K8s cluster
```
$ minikube start
$ minikube status

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
Install Jenkins using helm chart
```
$ helm repo add jenkins https://charts.jenkins.io
$ helm repo update
$ kubectl create namespace jenkins
$ helm install jenkins --namespace jenkins jenkins/jenkins --wait
$ kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
```
