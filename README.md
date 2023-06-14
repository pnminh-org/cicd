## Organization pipeline using centralized Jenkinsfile
When developing an application and utilizing Jenkins pipeline for CI/CD, it's common to have a Jenkinsfile stored in the same repository as the application. This allows easy configuration of the Jenkinsfile to meet the specific CI/CD requirements of the application.

However, in large organizations, allowing a custom Jenkinsfile in each application repository can have its drawbacks. These include:

Code vs. Jenkinsfile: The application team not only focuses on writing code that benefits users directly but also needs to ensure the quality and maintenance of the Jenkinsfile.

Cumbersome Support: Supporting the CI/CD process at the organizational level becomes challenging because each team may have its own unique approach to the process.

Enforcing Policies: Enforcing the company's policies for the CI/CD process, such as security scanning, becomes difficult as application teams are free to implement their own customized CI/CD workflows.

To address these challenges, organizations can consider using Jenkins shared libraries for their pipelines. However, this approach may still allow application teams to use their own Jenkinsfile and introduce undesired customizations.

To overcome these issues, the [Remote Jenkinsfile Provider](https://plugins.jenkins.io/remote-file/) plugin can be utilized. This plugin enables the usage of a centralized Jenkinsfile at the organization level, specifically within the "Organization folder" item in the Jenkins configuration. By adopting this approach, organizations can establish and enforce uniform CI/CD processes and policies across all applications throughout the organization, ensuring consistency and standardization.

This article provides a comprehensive guide on establishing an organization-wide CI/CD pipeline using a centralized Jenkinsfile. We will explore the step-by-step process, starting with setting it up through the user interface (UI), and then seamlessly transitioning those steps into everything-as-code configurations.

## Set up Jenkins server
### Installation options
To showcase the process outlined in this article, we will expediently configure a local Jenkins server for testing purposes. Among the plethora of [installation options](https://www.jenkins.io/doc/book/installing/) at your disposal, containerization emerges as an outstanding choice owing to its remarkable advantages:

* **Isolation and Portability**: Running Jenkins in a container provides a distinct environment, isolating it from the underlying host system. By encapsulating Jenkins and its dependencies within a container, it ensures consistent operation across different environments. This portability allows for easy migration and replication of the Jenkins setup, as the container can be deployed on any host system supporting the containerization platform.

* **Easy Deployment and Scalability**: Containerization streamlines the deployment process by having Jenkins and its dependencies packaged into a single container image, allowing for seamless deployment on any host system with the required container runtime. Furthermore, it facilitates effortless scaling by running multiple Jenkins instances, enabling horizontal scalability to handle increased workloads.

* **Version Management and Rollbacks**: Jenkins container images can be versioned and tagged, simplifying tracking and enabling easy rollback to previous versions if necessary.

* **Dependency Management**: Containerization offers a self-contained environment for Jenkins and its dependencies. This eliminates conflicts or compatibility issues with other applications or libraries present on the host system. Additionally, it enables efficient management and isolation of specific plugin or tool versions required by Jenkins, ensuring consistent and reproducible builds.

* **Resource Efficiency**: Containerization optimizes resource utilization. Jenkins containers can be configured with specific resource limits, such as CPU and memory, ensuring efficient utilization of system resources. This capability enables effective resource management, improved performance, and the ability to run multiple Jenkins instances on the same host without interference.

* **Easy Updates and Maintenance**: Containerization simplifies the process of updating or upgrading Jenkins. By updating the container image with the latest Jenkins version or applying patches, we can keep Jenkins up to date without impacting the host system. This streamlined approach to updates and maintenance tasks reduces the risk of disrupting the host environment or other applications running on it.

In this article, we will utilize [Red Hat OpenShift Local](https://developers.redhat.com/products/openshift-local), a lightweight single-node Openshift cluster designed for local computer resources, to deploy Jenkins. OpenShift offers the core features of Kubernetes while providing enhanced security, integrated developer experience, and additional enterprise-focused functionalities that meet the requirements of many companies and organizations.

You can follow this [link](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.5/html/getting_started_guide/installation_gsg) for step-by-step instructions on installing Red Hat OpenShift Local on your specific operating system platform.
### Installing Jenkins server using Helm chart
To simplify the installation process further, we will leverage a Jenkins Helm chart developed by CloudBees. The Helm chart enables rapid installation with convenient customization options for easy configuration.
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
