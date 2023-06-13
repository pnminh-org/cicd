## Organization pipeline using centralized Jenkinsfile
When developing an application and utilizing Jenkins pipeline for CI/CD, it's common to have a Jenkinsfile stored in the same repository as the application. This allows easy configuration of the Jenkinsfile to meet the specific CI/CD requirements of the application.

However, in large organizations, allowing a custom Jenkinsfile in each application repository can have its drawbacks. These include:

Code vs. Jenkinsfile: The application team not only focuses on writing code that benefits users directly but also needs to ensure the quality and maintenance of the Jenkinsfile.

Cumbersome Support: Supporting the CI/CD process at the organizational level becomes challenging because each team may have its own unique approach to the process.

Enforcing Policies: Enforcing the company's policies for the CI/CD process, such as security scanning, becomes difficult as application teams are free to implement their own customized CI/CD workflows.

To address these challenges, organizations can consider using Jenkins shared libraries for their pipelines. However, this approach may still allow application teams to use their own Jenkinsfile and introduce undesired customizations.

To overcome these issues, the [Remote Jenkinsfile Provider](https://plugins.jenkins.io/remote-file/) plugin can be utilized. This plugin enables the usage of a centralized Jenkinsfile at the organization level, specifically within the "Organization folder" item in the Jenkins configuration. By adopting this approach, organizations can establish and enforce uniform CI/CD processes and policies across all applications throughout the organization, ensuring consistency and standardization.

This article provides a comprehensive guide on establishing an organization-wide CI/CD pipeline using a centralized Jenkinsfile. We will explore the step-by-step process, starting with configuring it through the user interface (UI), and then seamlessly transitioning those steps into everything-as-code configurations.

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
 ![picture 15](images/85d32d723f005cb2fdea65c57ace75cf627ec45cbd28e53bb555a8ace211ec36.png)  
need `metadata` read access to scan repos
 ![picture 16](images/9cdc5460f18ecc81649ca7f54be3953b1aba34b0fede2907b5f597659af74865.png)  
need `contents` for branch scanning
![picture 17](images/444c9a66200d2945889afc5b032bb1878810bc526caff56fba7a8eae9dac9073.png)  
need `Pull requests` for PR scanning
![picture 18](images/5a8ef05681d8f360dc065ec58698a68f8e2d5a522c022942fe6a2d1cd65b5c2a.png)  

- Can choose classic github token
- Add github enterprise server
  ![picture 11](images/84078e9394cab60b9bbbb94848c896fbf823886dd939f99863ac12f670888ddf.png)  
- Configure org repos(creds are username/password)
![picture 12](images/715b6f02b3447aa8f967e35571985f4c6a46c5a11bfa42947093f7d6462b6bfc.png)  
- Allow Jenkinsfile from other repo to be used(need to restart Jenkins to have it take effect). Also requires [GitHub Branch Source plugin ](https://docs.cloudbees.com/docs/cloudbees-ci/latest/cloud-admin-guide/github-branch-source-plugin)
![picture 14](images/7fee139b23fd9a0f94326eaaa3543f3ddfb57f4f0af89c1df8b677a031228b4b.png)  
## Configuration as code
```
$ kubectl create namespace jenkins-iac
$ helm upgrade --install jenkins --namespace jenkins-iac jenkins/jenkins --values .helm/jenkins/values.yaml --values .helm/jenkins/creds-values.yaml --wait
```
