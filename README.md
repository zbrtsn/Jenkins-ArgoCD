# Jenkins-ArgoCD
Jenkins will pull the private GitLab repository. It will build the repository using the Dockerfile contained within the repo and push the image to Docker Hub.<br>
Then, we'll add the GitLab username and password to ArgoCD since the repository is private. We'll write and create the YAML file for the ArgoCD application setup within the pipeline.<br>
Finally, we'll issue the command to create the ArgoCD application with this YAML file and complete the process.

## Explanation

```
        environment {
        registryCredential = 'dockerhub-credentials' // Holds the username and password credentials for logging in to Docker Hub
        GIT_REPO_URL = 'https://mtgit.mediatriple.net/zubeyir.tosun/private-basic-to-do-flask.git' // GitLab or GitHub repository URL
        DOCKERHUB_REPO = 'pyouck/basic_todo_flask' // Docker Hub repository URL
        ARGOCD_SERVER = '127.0.0.1:35883' // ArgoCD server address
        ARGOCD_APP = 'app-flask' // Name of the ArgoCD application. It also creates a namespace with this name. You can change it in the YAML file below if needed.
        KUBECONFIG = credentials('kubeconfig-file') // The ID of the kubeconfig file you uploaded. This allows Jenkins to connect to Kubernetes. I think if your Jenkins works on your kubernetes then you don't need to do that.
        KUBE_SERVICE = 'https://kubernetes.default.svc' // The URL of the target Kubernetes cluster
        }
```
This is where the global variables are located. You can make general changes from here. You may need to make a few changes within the code as well, which are detailed in comment sections within this part of the code.<br>
#### Kubeconfig
If you have installed Jenkins inside Kubernetes or Kubernetes outside of your local environment, you do not need to introduce the Kubernetes config file to Jenkins. It is especially necessary to specify it if your Kubernetes is local. If you are encountering a Groove language error, it might be due to Jenkins not being able to see the config file and thus not being able to communicate with Kubernetes.
<br>
#### ArgoCD Server
Don't write it as 'http' or 'https'; provide it directly.


<br><br>

```
        stage('Clone Git Repository') { // The stage where the Git project is cloned
            steps {
                git branch: 'latest', credentialsId: 'gitlab-login', url: "${GIT_REPO_URL}"
            }
        }
```
If you have reviewed my previous codes, you would know that we cloned the repo differently. The purpose of pulling the repo this way is due to the Git repo being private. The credential 'gitlab-login' we provide in the middle of the code contains the GitLab or GitHub username and password. Also, don't forget to change the repo's branch and repo URL.

<br><br>

```
        stage('Build Docker Image') { // The stage where the project pulled from the repo is built
            steps {
                script {
                    dockerImage = docker.build("${DOCKERHUB_REPO}:test2", "--no-cache .")
                }
            }
        }
```
It builds the repository that it pulls.

<br><br>

```
        stage('Push Docker Image to DockerHub') { // Push to Docker Hub
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("test2")
                    }
                }
            }
        }
```
Here, registryCredential contains the Docker Hub account username and password under the credential named dockerhub-credentials.<br>
This allows the docker.withRegistry code to connect to Docker Hub, which is pre-installed with a Jenkins plugin. After logging in, it pushes the built image with the "test2" tag to the name specified in DOCKERHUB_REPO.

<br><br>

```
        stage('Check and Manage Secret') { // Adds the GitLab repository to ArgoCD. Since the repo is private, we need to establish the connection in between.
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitlab-login', usernameVariable: 'GITLAB_USERNAME', passwordVariable: 'GITLAB_PASSWORD')]) {
                        sh """
                        argocd repo add $GIT_REPO_URL \
                          --username \$GITLAB_USERNAME \
                          --password \$GITLAB_PASSWORD
                          """

                    }
                }
            }
        }
```

<br><br>

```
        stage('Create ArgoCD Application YAML') { // Creates the YAML file to define the new application in ArgoCD
            steps {
                script {
                    def yamlContent = """
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ${ARGOCD_APP}
  namespace: argocd
spec:
  project: default
  source:
    repoURL: ${GIT_REPO_URL}
    path: .    # The project files are located in the root of the repo, so we get everything without specifying a path.
    targetRevision: HEAD
    directory:
      recurse: true
  destination:
    server: ${KUBE_SERVICE}
    namespace: ${ARGOCD_APP}
  syncPolicy:
    automated:   # Enables ArgoCD to automatically synchronize
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true     # Allows creating the ${ARGOCD_APP} namespace if it does not already exist.
                    """
                    
                    writeFile file: 'argocd-application.yaml', text: yamlContent // Writes the above content to a file named "argocd-application.yaml"
                    
                }
            }
        }
```
The reason I don't want to write the YAML file directly here is to avoid having to manage tasks like pushing to Git and making updates in separate windows, which can be slow.

Additionally, I pulled the variables that need to be changed from the environment section to minimize the need for extensive modifications in the code during the editing phase. <br>

I made the application name and namespace in ArgoCD the same. You can change the namespace name in this YAML section as needed, or create a variable in the environment section with the name and use that variable here.<br>

`CreateNamespace=true` <br>
This ensures that a namespace is created if it doesn’t already exist with the specified name.<br>

------------
## Related Projects

Here are some related projects;

[Basic Flask Project](https://github.com/zbrtsn/basic-to-do-flask.git) <br>
The same project as in the code (GIT_REPO_URL), but the manifest (YAML) files are missing. <br><br>
[Kubernetes-Kind](https://github.com/zbrtsn/kurbernetes-kind) <br>
The YAML file should be similar to the one here.<br>
The YAML files and the project files are in the same directory.


