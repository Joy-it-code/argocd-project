# GitOps Configuration Management with ArgoCD

> This project demonstrates advanced configuration management in Kubernetes application deployments using **ArgoCD**, **Helm**, **Kustomize** and **external secret management** tools like **HashiCorp Vault** and **AWS Secrets Manager**.


```bash
my-argocd-project/
├── my-app/                    
│   ├── Chart.yaml             
│   ├── values.yaml            
│   └── templates/             
│       ├── deployment.yaml    
│       └── service.yaml      
├── my-app-kustomize/         
│   ├── base/                 
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── myresource.yaml    
│   │   └── kustomization.yaml
│   └── overlays/
│       └── dev/               
│           ├── patch.yaml     
│           └── kustomization.yaml
├── helm-app.yaml              
├── kustomize-app.yaml         
├── myresource-crd.yaml        
└── argocd-cm.yaml
```

## Prerequisites

- Make sure you have the following installed:

- A computer with Git, kubectl, Helm, and ArgoCD CLI installed.
- A Kubernetes cluster (like Minikube or a cloud-based one).
- A Git repository (e.g., on GitHub) to store your files.
- ArgoCD installed on your Kubernetes cluster.


## Project Setup


## 1: Check Installed Tools
```bash
git --version
helm version
kubectl version --client
argocd version --client
```
![](./img/1d.installation.version.png)



### Start Minikube and Check Kubernetes Cluster
```bash
minikube start
kubectl cluster-info
```


## Install ArgoCD in the cluster:

- Install ArgoCD in Kubernetes cluster and Verify:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```



### Log in to ArgoCD:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
argocd login localhost:8080 --username admin --password <paste-the-password>
```


- Forward the ArgoCD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

- Open `http://localhost:8080` on browser.
**Enter admin and the password when prompted.**


### Test:
```bash
kubectl get pods -n argocd
```
![](./img/1e.get.pod.argocd.png)




## Step 2: Create a Git Repository

##  Create a new folder for the Project:
```bash
mkdir argocd-project
cd argocd-project
git init
```


### Add and Push Folder to Github Repository 

```bash
git add .
git commit -m "initial commit"
git branch -M main
git remote add origin https://github.com/your-username/argocd-project.git
git push -u origin main
```



## Create a Helm Chart
```bash
helm create my-app
```

## Simplify the Helm chart

- Edit `my-app/Chart.yaml`
```bash
apiVersion: v2
   name: my-app
   description: A simple Helm chart for a toy app
   version: 0.1.0
```


- Edit `my-app/values.yaml`
```bash
replicaCount: 1

   image:
     repository: nginx
     tag: "latest"
     pullPolicy: IfNotPresent

   service:
     type: ClusterIP
     port: 80

   ingress:
     enabled: false
```




### Edit the service in `my-app/templates/deployment.yaml`

```bash
apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: {{ .Release.Name }}
   spec:
     replicas: {{ .Values.replicaCount }}
     selector:
       matchLabels:
         app: {{ .Release.Name }}
     template:
       metadata:
         labels:
           app: {{ .Release.Name }}
       spec:
         containers:
         - name: my-app
           image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
           imagePullPolicy: {{ .Values.image.pullPolicy }}
           ports:
           - containerPort: 80
```



### Edit the service in `my-app/templates/service.yaml`:

```bash
nano my-app/templates/service.yaml
```

### Paste:
```bash
apiVersion: v1
   kind: Service
   metadata:
     name: {{ .Release.Name }}
   spec:
     selector:
       app: {{ .Release.Name }}
     ports:
     - port: {{ .Values.service.port }}
       targetPort: 80
     type: {{ .Values.service.type }}
```


### Test Helm Chart Locally:
```bash
helm lint my-app
```


### Push the Helm chart to Git:

```bash
git add my-app
git commit -m "Add Helm chart for my-app"
git push
```

### Test:
- Run a Helm dry-run to check the chart:
```bash
helm template my-app ./my-app
```


## 3: Deploy the Helm Chart with ArgoCD

- Create an ArgoCD application for the Helm chart:
```bash
nano helm-app.yaml
```

```bash
apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app-helm
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/your-username/my-gitops-project.git
       path: my-app
       targetRevision: main
     destination:
       server: https://kubernetes.default.svc
       namespace: default
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
```

### Apply the application and Check ArgoCD::
```bash
kubectl apply -f helm-app.yaml
argocd app list
argocd app get my-app-helm
```
![](./img/2a.kube.apply.png)
![](./img/2b.argocd.list.png)


or visit `http://localhost:8080` on Browser

![](./img/2c.health.page1..png)
![](./img/2d.sync.page1.png)


### Check the deployment and service:
```bash
kubectl get deployments
kubectl get services
```
![](./img/2e.get.svc.deplymt.png)



## 4: Create a Kustomize Configuration

- Create a Kustomize base:
```bash
mkdir -p my-app-kustomize/base
```


### Create `my-app-kustomize/base/deployment.yaml` file

#### Paste:
```bash
apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-app
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: my-app
     template:
       metadata:
         labels:
           app: my-app
       spec:
         containers:
         - name: my-app
           image: nginx:latest
           ports:
           - containerPort: 80
```


### Create `my-app-kustomize/base/service.yaml` File

#### Paste:
```bash
apiVersion: v1
   kind: Service
   metadata:
     name: my-app
   spec:
     selector:
       app: my-app
     ports:
     - port: 80
       targetPort: 80
     type: ClusterIP
```


### Create `my-app-kustomize/base/kustomization.yaml` File

#### Paste:
```bash
apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
   - deployment.yaml
   - service.yaml
```



### Create a dev overlay:

```bash
mkdir -p my-app-kustomize/overlays/dev
```

### Create `my-app-kustomize/overlays/dev/patch.yaml` File


#### Paste:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
    spec:
     replicas: 2
```

### Create `my-app-kustomize/overlays/dev/kustomization.yaml` File

#### Paste:
```bash
resources:
  - ../../base
patches:
  - path: patch.yaml
    target:
      kind: Deployment
      name: my-app
```


Push to Git:
```bash
git add my-app-kustomize
git commit -m "Add Kustomize configuration"
git push 
```

### Test dev overlay:
```bash
kubectl kustomize my-app-kustomize/overlays/dev
```


## 5: Deploy the Kustomize Configuration with ArgoCD

- Create an ArgoCD application for Kustomize:
```bash
touch kustomize-app.yaml
```

#### Paste:
```bash
apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app-kustomize
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/your-username/argocd-project.git
       path: my-app-kustomize/overlays/dev
       targetRevision: main
     destination:
       server: https://kubernetes.default.svc
       namespace: default
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
```         


### Apply the application:
```bash
kubectl apply -f kustomize-app.yaml
```

### Check ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd app list
argocd app get my-app-kustomize
```