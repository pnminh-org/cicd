## Organization pipeline using centralized Jenkinsfile
When developing an application and utilizing Jenkins pipeline for CI/CD, it's common to have a Jenkinsfile stored in the same repository as the application. This allows easy configuration of the Jenkinsfile to meet the specific CI/CD requirements of the application.

However, in large organizations, allowing a custom Jenkinsfile in each application repository can have its drawbacks. These include:

* **Code vs. Jenkinsfile**: The application team must balance their focus on writing code for users with maintaining the quality and upkeep of the Jenkinsfile.

* **Cumbersome support**: Supporting organization-wide CI/CD process becomes challenging with teams using unique approaches.

* **Policy compliance**: Enforcing company policies for the CI/CD process, such as security scanning, becomes difficult as application teams are free to implement their own customized workflows.

To tackle these challenges, Jenkins shared libraries can be used, but this approach may still allow application teams to utilize their own Jenkinsfile and introduce undesired customizations.

The [Remote Jenkinsfile Provider](https://plugins.jenkins.io/remote-file/) plugin may offer a solution to those issues. It enables a centralized Jenkinsfile with the `Organization Folder` and `MultiBranch` pipeline options in Jenkins. By adopting this approach, organizations can establish and enforce uniform CI/CD processes and policies across development teams, ensuring consistency and standardization.

This article provides a comprehensive guide on establishing an organization-wide CI/CD pipeline using a centralized Jenkinsfile. We'll cover the step-by-step process, starting with UI setup and transitioning seamlessly to everything-as-code configurations.

## Set up Jenkins server
### Installation options
We will use a local Jenkins server to showcase the process outlined in this article. Among many [installation options](https://www.jenkins.io/doc/book/installing/), containerization emerges as an outstanding choice owing to its remarkable advantages:

* **Isolation and Portability**: Running Jenkins in a container ensures consistent operation across different environments, allowing easy migration and replication on any host system supporting the containerization platform.

* **Easy Deployment and Scalability**: Containerization simplifies deployment with a single container image and enables effortless scaling to handle increased workloads.

* **Version Management and Rollbacks**: Jenkins container images can be versioned and tagged, facilitating easy tracking and rollback to previous versions.

* **Dependency Management**: Containerization provides a self-contained environment, eliminating conflicts with other applications and allowing for efficient management of Jenkins dependencies.

* **Resource Efficiency**: Containers can be configured with specific resource limits, optimizing resource utilization and enabling multiple Jenkins instances on the same host.

* **Easy Updates and Maintenance**: Updating Jenkins becomes straightforward by updating the container image, reducing the risk of disrupting the host environment or other applications.

[Red Hat OpenShift Local](https://developers.redhat.com/products/openshift-local), a lightweight single-node Openshift cluster,  is an excellent choice for deploying our Jenkins containers. OpenShift provides Kubernetes core features with enhanced security, integrated developer experience, and enterprise-focused functionalities. Follow this [link](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.5/html/getting_started_guide/installation_gsg) for step-by-step installation instructions tailored to your operating system.

### Installing Jenkins server using Helm chart
To simplify the installation process further, we will leverage the [official helm chart](https://artifacthub.io/packages/helm/jenkinsci/jenkins) developed by Jenkins. This helm chart enables rapid installation with convenient customization options for easy configuration. We use the default configs for the first iteration.
Start with creating a project for Jenkins
```bash
$ oc new-project jenkins
```
By default OCP run pods with non-privileged access to mitigate risks, restrict capabilities, and prevent harmful actions within the cluster, so we need a helm values file for Jenkins that allow us to use a non-root user:

```yaml
controller:
  podSecurityContextOverride: #required for pod running on OCP
    runAsNonRoot: true
    runAsUser: 1000650000
    runAsGroup: 1000650000
  containerSecurityContext: #required for jenkins init container
    runAsUser: 1000650000
    runAsGroup: 1000650000
  initContainerEnv: #https://github.com/jenkinsci/helm-charts/issues/506
    - name: CACHE_DIR
      value: "/tmp/cache"
```

The values for `runAsUser` and `runAsGroup` can be found from the Openshift project's annotations:
```bash
oc get project/jenkins -o jsonpath='uid-range:{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}{"\n"}supplement-groups:{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}{"\n"}' 
uid-range:1000670000/10000
supplement-groups:1000670000/10000
```
You can read more about Openshift's `uid` and `gid` [here](https://cloud.redhat.com/blog/a-guide-to-openshift-and-uids)

**Note** The plugins that are specified with `controller.installPlugins` in the default [helm values](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/values.yaml) may not up-to-date or have conflicting dependency versions with other plugins so it may be better to specify specific version in our custom values file as well:
```yaml
controller:
  ...
  installPlugins:
    - kubernetes:3900.va_dce992317b_4
    - workflow-aggregator:596.v8c21c963d92d
    - configuration-as-code:1647.ve39ca_b_829b_42
    - git:5.1.0
```
In the future article, we will look into creating custom Jenkins image that has all the needed dependencies to avoid any conflicting versions and to speed up the startup time for Jenkins server.

We then can install Jenkins using helm:
```bash
$ helm upgrade --install jenkins jenkins/jenkins --values .helm/jenkins/values.yaml
```
### Installing required plugins
We can access the Jenkins UI at http://localhost:8080 via port-forwarding.
```bash
$ oc --namespace jenkins port-forward svc/jenkins 8080:8080
```
Log in with the admin user and the password generated during Jenkins installation.

Navigate to `Manage Jenkins > Plugins > Available plugins` and install the following plugins:

![Install Plugin](images/install_plugins.png)  


- [GitHub](https://plugins.jenkins.io/github/): integrates Jenkins with Github projects as we will use Github as our example   
- [GitHub Branch Source](https://plugins.jenkins.io/github-branch-source/): integrate `Organization Folder` and `Multibranch Pipeline`  items with GitHub
- [Remote Jenkinsfile Provider](https://plugins.jenkins.io/remote-file/): allow using a centralized Jenkinsfile from a remote repository
- [GitHub API](https://plugins.jenkins.io/github-api/): dependency for other GitHub related plugins.

Make sure Jenkins is restarted after all the plugins are installed to have them work properly. 

### Set up GitHub Organization and access token
GitHub organizations facilitate collaborative work, empowering teams to manage repositories, access permissions, and establish hierarchical structures. Personal accounts, on the other hand, are meant for individual profiles with limited collaboration features. To create a new organization, navigate to your `Personal account's Settings > Organizations > New organization`:

![Create Organization](images/create_org.png) 


In addition, to facilitate communication between the Jenkins server and our GitHub organization, it is necessary to create a GitHub access token. Visit `Personal account's Settings > Developer settings > Personal access tokens > Fine-grained tokens` to generate a new token. Although classic tokens are available, fine-grained tokens offer more granular access control.

Make sure the `Resource owner` is set to your organization account:
![GitHub access token](images/github_access_token.png)  

For our pipeline to function correctly with the installed plugins, `read-only` access is necessary for the following GitHub organization permissions: `Contents`, `Metadata`, and `Pull requests`. If you want Jenkins to modify data, write permissions will be required as well. Remember to save the generated token securely, as GitHub displays it only once.
 

### Configure Organization Folder item
Go to `Jenkins UI > New item` and create a `Organization Folder` pipeline.
![Organization Folder Pipeline](images/org_folder_item.png)  


Fill in descriptions for your organization in `General` section. 

For `Projects`, choose `GitHub Organization`. Leave `API endpoint` empty unless you have a custom server for your org. For `Credentials`, we will add the access token generated from the previous step. Click `Add` to add your credentials, either at `global` scope or at your organization folder pipeline one.
![Jenkins Creds](images/add_jenkins_creds.png)  
Use the GitHub organization account name for `Owner` field.
The configurations should look like this:
![Configure Org Projects](images/configure_org_projects.png)  

Regarding `Behaviors`, you can select how Jenkins check out repositories, branches and PRs for builds.
Apart from default behaviors, we will filter the repositories based on their topic names by selecting `Filter by Repository Topics` option. Only those with the topics mentioned will be added to the organization folder
![Filter Repo Topics](images/filter_repo_topics.png)  

For `Project Recognizers`, as we will use a remote Jenkinsfile, let's remove `Pipeline Jenkinsfile` option.
With the remote Jenkinsfile, I will use the one on [my GitHub organization](https://github.com/pnminh-org/cicd) at `jenkins/org-pipeline/Jenkinsfile`.
![Configure Remote Jenkinsfile](images/configure_remote_jenkinsfile.png)

Leave others as-is and save the configuration. 

### Scanning repositories and run the first pipeline
As mentioned early, we only add repositories with the `topics` matching the configuration for our `organization folder`, in this case `cicd`. Let's create a new repository under our organization and add `cicd` topic to it.
![Test App repository](images/test_app_repo.png)  

Now go back to `Jenkins UI Dashboard> <Your Org folder name> > Scan Organization Now`. The new repository should show up after the scan.
![Test App pipeline](images/test_app_pipeline.png)  
When we run a build, the remote Jenkinsfile will also be checked out and used for the pipeline:
![Pipeline Logs](images/pipeline_logs.png)  



### Jenkins configuration as code
Instead of going through the Jenkins UI to set up our `Organization folder` structure, Jenkins provides [Jenkins Configuration as Code plugin](https://www.jenkins.io/projects/jcasc/) to configure Jenkins pipelines in a fully reproducible way. 
Let's first add the additional plugins we need for our pipeline to the `values.yaml` file
```yaml
controller:
  ...
  additionalPlugins:
    - github-api:1.314-431.v78d72a_3fe4c3
    - github:1.37.1
    - github-branch-source:1725.vd391eef681a_e #for Github repo structure 
    - remote-file:1.23 #for remote Jenkinsfile
    - job-dsl:1.84 #for org folder setup
```

Then add the GitHub access token to Jenkins. create a file, `creds-values.yaml`
```yaml
controller:
  JCasC:
    configScripts:
      global-creds: |
        credentials:
          system:
            domainCredentials:
              - credentials:
                  - usernamePassword:
                      scope: GLOBAL
                      id: "pnminh-org"
                      username: "pnminh"
                      password: <GITHUB_ACCESS_TOKEN>
                      description: "Username/Password Credentials for pnminh-org authentication"
```
We then can create a [Job DSL](https://plugins.jenkins.io/job-dsl). If you are not familiar with Job DSL syntax, [here](https://plugins.jenkins.io/job-dsl/#plugin-content-getting-started) is a quick way to test your the script before make it into a working version. Also the `Job DSL API reference` located at https://<your.jenkins.installation>/plugin/job-dsl/api-viewer/index.html should give you the idea of what need to be added to the script.  Here is the one, `org-folder-job-dsl-values.yaml`, that will configure our `Organization folder` in the same way we did with the UI:
```yaml
controller:
  JCasC:
    configScripts:
      job-dsl: | #file name, can be any name. The content will be picked up by Jenkins
        jobs:
          - script: |
              organizationFolder('pnminh-org') {
                description('This contains branch source jobs for Bitbucket and GitHub')
                displayName('pnminh-org')
                triggers {
                  cron('@midnight')
                }
                organizations {
                  github {
                    repoOwner("pnminh-org")
                    credentialsId("pnminh-org")
                    traits {
                      // Discovers branches on the repository.
                      gitHubBranchDiscovery {
                        // Determines which branches are discovered.
                        // 1 = Exclude branches that are also filed as PRs
                        strategyId(1)
                      }
                      gitHubPullRequestDiscovery {
                        // 1 = Merging the pull request with the current target branch revision
                        strategyId(1)
                      }
                      // Only scan repos that have topics
                      gitHubTopicsFilter {
                        topicList("cicd")
                      }
                      gitHubExcludeArchivedRepositories()
                    }
                  }
                }
                configure {
                  def traits = it / navigators / 'org.jenkinsci.plugins.github__branch__source.GitHubSCMNavigator' / 'traits'
                  traits << 'org.jenkinsci.plugins.github__branch__source.ForkPullRequestDiscoveryTrait' {
                      strategyId(1)
                      trust(class: 'org.jenkinsci.plugins.github_branch_source.ForkPullRequestDiscoveryTrait$TrustEveryone')
                  }                 
                }
                projectFactories {
                  remoteJenkinsFileWorkflowMultiBranchProjectFactory {
                    // File or directory name used as marker to recognize the project need to be build.
                    localMarker(null)
                    remoteJenkinsFileSCM {
                      // The git plugin provides fundamental git operations for Jenkins projects.
                      gitSCM {
                        userRemoteConfigs {
                          userRemoteConfig {
                            // Specify the URL or path of the git repository.
                            url("https://github.com/pnminh-org/cicd")
                            // ID of the repository, such as origin, to uniquely identify this repository among other remote repositories.
                            name("origin")
                            // A refspec controls the remote refs to be retrieved and how they map to local refs.
                            refspec("main")
                            // Credential used to check out sources.
                            credentialsId("pnminh-org")
                          }
                        }
                        // List of branches to build.
                        branches {
                          branchSpec {
                          // Specify the branches if you'd like to track a specific branch in a repository.
                            name("main")
                          }
                        }
                        browser {
                          github {
                            // Specify the HTTP URL for this repository's GitHub page (such as https://github.com/jquery/jquery).
                            repoUrl("https://github.com/pnminh-org/cicd")
                          }
                        }
                        gitTool(null)
                      }
                    }
                    matchBranches(false)
                    remoteJenkinsFile("jenkins/org-pipeline/Jenkinsfile")
                  }
                }       
              }
```
**Note**: as the time of writing, I could not use the Job DSL API script for `ForkPullRequestDiscoveryTrait` as part of `GitHub Branch Source` plugin. Using [configure block](https://github.com/jenkinsci/job-dsl-plugin/wiki/The-Configure-Block), and the path retrieved from `$JENKINS_HOME/jobs/<Organization folder name>/config.xml` did the trick for me. More info of the issue can be found here `https://issues.jenkins.io/browse/JENKINS-61119`
Run a new installation of Jenkins using all the values files we just created/updated:
```bash
$ oc new-project jenkins-jcasc
$ helm upgrade --install jenkins jenkins/jenkins --values .helm/jenkins/values.yaml --values .helm/jenkins/creds-values.yaml --values .helm/jenkins/org-folder-job-dsl-values.yaml
```
