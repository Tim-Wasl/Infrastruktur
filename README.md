# Infrastruktur
## Minikube start
Start Minikube with more work memory, with default it is possible that the elasticsearch instance in SonarQube fails. Then change the profile to gitops.
```
minikube start -p gitops --memory 8192 --cpus 4
minikube profile gitops
```
## Setup
Fork the Infrastruktur and Code repository (https://github.com/Tim-Wasl/Code).




## Argo CD
Create the namespace argocd and apply the argo manifest.
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Port forward the argo-server and get the password from minikube dashboard -> namespace argocd -> secrets -> argocd-initial-admin-secret.
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
## App
Change the repoURL in the deployment.yaml file in the folder argo-cd to point to your forked repository.Then create the namespace for the app and apply the argo-cd deployment.yaml.
```
kubectl create namespace app
kubectl apply -f ./argo-cd/deployment.yaml
```
Now go to the argo dashboard on port 8080 and see the running app. To get access to the endpoints, forward the gitops-service and you can access the app on localhost with theminikube given port and path /hello.
```
minikube service -n app gitops-service
```
## Secrets

### Install sealed Secret
 First of all delete the selaed secrets in docker-credentials, tekton/sonarqube, tekton/github-webhook in your forked Infrastruktur repository. Now add the sealed secret controller and install kubeseal according to the doku:
 https://github.com/bitnami-labs/sealed-secrets
 For Windows you have to download the windows-tar.gz file, unzip it and add the path to the kubeseal folder in your environment variable for Path.
 ```
 kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.23.0/controller.yaml
 ```

 ### Github-Secret
 To create your github secret got to your .ssh folder and base 64 encode the ir_rsa and the known_hosts file and create an yaml file with the folloing content.
 !!! Dont push this secret to github !!!
 ```
 kind: Secret
apiVersion: v1
metadata:
  name: github-secret
  namespace: ci
data:
  id_rsa:
  known_hosts:
 ```
 Seal the secret with the following command (windows):
 ```
 Get-Content .\github-secret.yaml | kubeseal --format=yaml > .\github-sealed.yaml  
 ```
## DockerHub-Secret
 In the folder docker-credentials add your dockerhub username and password in the docker-config.conf file. Then base 64 encode this file and create a secret:

 ```
apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
  namespace: ci
data:
    config.json: 
 ```

 Now seal this secret too:
 ```
 Get-Content .\docker-secret.yaml | kubeseal --format=yaml > .\docker-sealed.yaml 
 ```
 ## Tekton
 In triggertemplate change the value of the repourl to you forked code repository. In the pipeline.yaml change in the task build-push the value of the dockerhub url. In the last task change the remote origin to your forked infrastructure repository and in the sed Befehl add your dockerhub repository name.

 ### Add Tekton Resources
 Apply the tekon resources and acces the dashboard via port forwarding:
 ```
 kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release-full.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
 ```

Create the ci namespace and apply the tekton tasks:
 ```
kubectl create namespace ci

kubectl apply --namespace=ci -f https://raw.githubusercontent.com/tektoncd/catalog/master/task/git-clone/0.9/git-clone.yaml

kubectl apply --namespace=ci -f https://api.hub.tekton.dev/v1/resource/tekton/task/gradle/0.3/raw

kubectl apply --namespace=ci -f https://api.hub.tekton.dev/v1/resource/tekton/task/kaniko/0.6/raw

kubectl apply --namespace=ci -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-cli/0.4/git-cli.yaml

 ```

 ### Argo CI Deployment
 Change the repoUrl in the argo-cd ci-deployment.yaml file to your forked infrastructure repository. Push all changes made and then apply the ci-deployment.yaml file. Then make the eventlistener accessible using ngrok.
 ```
 minikube service -n ci el-event-listener
 ngrok http <first port from evl>
 ``` 

Bevor sending a webhook from the code repo to the ngrok address (<ngrok-addresse/github-webhook/>) configure a sonarqube secret. 
```
minikube service -n ci sonarqube-service
```

Login with admin / admin, create a token in your profile and add it not base 64 encoded to a secret file.

```
apiVersion: v1
kind: Secret
metadata:
  name: sonarqube-secret
  namespace: ci
type: Opaque
stringData:
  sonar-token-key:
```
The last step before running the pipeline is, to configure argo cd. In the ui go to projects > namespace resource deny list, add TaskRun and PipelineRun in Group tekton.dev.