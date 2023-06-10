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
![picture 1](images/a5359b01d2ea287fb71ea8a755874c9e56c7434031147717355fde81230ca7c8.png)  
![picture 2](images/0a3529bc993366a821d55f2022a36be1a2a3e96dde1fcf909b1bbf0a7b486d3f.png)  
![picture 3](images/84971ced0ef2074601433e3ea611acd13f0171ebf92abefb9bca97e731eec78a.png)  
![picture 4](images/5b145d03f9cb10c5a0cfc8106b8a7f7c341f5b6df012043919f2305293b59d3d.png)  
![picture 5](images/2fa646854be9d64270335edcf277ddbf410541ffe1c324bfb96b37b6747e78af.png)  
![picture 6](images/84918d1df2a2c5282fe9394bec67621ea63c01bc31f9ec420044995841880855.png)  
![picture 7](images/a880b1f853f3a58cd641af85632b382f33f052b2d39ff300a0f2af584ba8d624.png)  
Install Github plugins for Jenkins
![picture 13](images/5283513b8734605ea5668831f4d0c5eb1f1382cfc1b426cff1eda6ad7a357c5a.png)  

Get Org API endpoint
- Go to org > Settings > Developer Settings > OAuth Apps
  ![picture 8](images/573db7acb3cba7271b972dedf463979f8bae160d3cab9cfb0fb121a09491cf2a.png)  

- Go to Org > Settings > ![picture 9](images/f1af019560a21aa84752dbefb7184da590eb9387728cd2decbe53eb326c685cb.png)  
- Go to User account > Developer settings > Personal access tokens > Fine grained tokens
![picture 10](images/a2101f088859184e4a5fccdfaac92f2225f8c3899c2016a93b19c77ac45e3b17.png)  
- Can choose classic github token
- Add github enterprise server
  ![picture 11](images/84078e9394cab60b9bbbb94848c896fbf823886dd939f99863ac12f670888ddf.png)  
- Configure org repos(creds are username/password)
![picture 12](images/715b6f02b3447aa8f967e35571985f4c6a46c5a11bfa42947093f7d6462b6bfc.png)  
- Allow Jenkinsfile from other repo to be used(need to restart Jenkins to have it take effect). Also requires [GitHub Branch Source plugin ](https://docs.cloudbees.com/docs/cloudbees-ci/latest/cloud-admin-guide/github-branch-source-plugin)
![picture 14](images/7fee139b23fd9a0f94326eaaa3543f3ddfb57f4f0af89c1df8b677a031228b4b.png)  
- Full configuration:
<video src='images/fullscreen.mov' width=180/>