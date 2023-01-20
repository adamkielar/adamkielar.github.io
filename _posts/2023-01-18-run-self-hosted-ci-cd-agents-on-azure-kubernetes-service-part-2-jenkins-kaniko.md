---
layout: post
title: "Run self-hosted CI/CD agents on Azure Kubernetes Service - Part 2 - Jenkins + Kaniko"
tags: ["azure", "aks", "jenkins", "kubernetes", "grafana", "promethes", "kaniko"]
---

![Diagram](/assets/post5/jenkins-diagram.png)

**This post is a continuation of our journey with self-hosted CI/CD agents. I encourage you to check [part 1](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-1-circleci-buildah/){:target="_blank"} if you want to see a different approach to that topic. In this post we will focus on [Jenkins](https://www.jenkins.io/){:target="_blank"}. It took a bit of time to install Jenkins on AKS. I encountered a few errors along the way. I will share my solution here so it might be helpful to others.**

## Overview

1. Pros
- Highly configurable with plugins and extensions
- Active user base
- Provides APIs

2. Cons
- Only community support
- Difficult to configure
- Difficult to debug
- Documentation could be improved

We will create and configure the following resources to show what Jenkins can offer:
- AKS cluster
- Azure Container Registry
- Promethes and Grafana to observe Jenkins agents
- Jenkins
- Github repo and connect it with Jenkins

<blockquote class="prompt-info">
<a href="https://github.com/adamkielar/jenkins-runner" target="_blank">Here is a link to GitHub repo with all files for reference</a>
. I created a script with the necessary commands to provision a basic setup. In this setup, we will use our own kubelet managed identity. I want to show you other possibilities for how we can create a cluster and connect other resources. We will assign AcrPull role to that identity by adding --attach-acr to az aks create command.
</blockquote>

## Create AKS cluster and ACR

1. Run script:
```bash
# After running above script, if there were no errors, variables should be available in terminal.
chmod +x aks.sh
./aks.sh 'add user id, for me, it is my email of AAD user'
```

2. Get credentials to AKS and test connection:

```bash
az aks get-credentials --resource-group $resourceGroup --name $aksName
kubectl get nodes
```

3. Create Jenkins namespace:

```bash
kubectl create namespace jenkins
```

4. Create Service Principal with AcrPush role to ACR. We will use this credentials in Jenkins pipeline to push image. In previous posts we investigated [login/password](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-1-circleci-buildah/#create-azure-container-registry){:target="_blank"} method and also [managed identity](https://www.adamkielar.pl/posts/how-to-use-an-azure-ad-workload-identity-on-azure-kubernetes-service/){:target="_blank"}. 

```bash
az ad sp create-for-rbac -n jenkinsAcrAccess --role AcrPush --scope $acrId
```

5. Save Service Principal credentials as Kubernetes Secret. Add your credentials to the command:

```bash
kubectl -n jenkins create secret generic acr-sp \
--from-literal=AZURE_CLIENT_ID=<appId> \
--from-literal=AZURE_CLIENT_SECRET=<password> \
--from-literal=AZURE_TENANT_ID=<tenant>
```

6. Create ConfigMap from config.json file with information about container registry. We will use it in Jenkins pipeline to inform Kaniko about ACR.
File the name with your ACR name (`echo $acrName`):

```json
{ "credHelpers": { "<ACR name>.azurecr.io": "acr-env" } }
```

```bash
kubectl -n jenkins create configmap docker-config --from-file=config.json
```

## Install Prometheus and Grafana

We will observe how many resources Jenkins and agent consume. We will use the prometheus-community Helm chart.

1. Add Helm repository:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
2. Install Helm chart in monitoring namespace:
```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```
3. Confirm that resources are running:
```bash
kubectl get all -n monitoring
```
4. Expose Grafana and Prometheus in two tabs in terminal
```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 4000:80
```
```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 4001:9090
```
5. Log in to Grafana. Default login/password: `admin/prom-operator`
```bash
open http://localhost:4000/login
```

![Grafana empty](/assets/post5/grafana-1.png)

**We will come back here once we will install Jenkins.**

## Install Jenkins

We will install Jenkins using Helm chart.

1. Add Helm repository:
```bash
helm repo add jenkinsci https://charts.jenkins.io
helm repo update
```
2. Create Service Account and Cluster Role:
```yaml
# jenkins-sa.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
rules:
- apiGroups:
  - '*'
  resources:
  - statefulsets
  - services
  - replicationcontrollers
  - replicasets
  - podtemplates
  - podsecuritypolicies
  - pods
  - pods/log
  - pods/exec
  - podpreset
  - poddisruptionbudget
  - persistentvolumes
  - persistentvolumeclaims
  - jobs
  - endpoints
  - deployments
  - deployments/scale
  - daemonsets
  - cronjobs
  - configmaps
  - namespaces
  - events
  - secrets
  verbs:
  - create
  - get
  - watch
  - delete
  - list
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:jenkins
```
```bash
kubectl apply -f jenkins-sa.yaml
```
3. Create Storage Class with custom mount options:
```yaml
# jenkins-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: jenkins-azurefile
provisioner: file.csi.azure.com
mountOptions:
  - uid=1000
  - gid=1000
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
parameters:
  skuName: Standard_LRS
```
```bash
kubectl apply -f jenkins-storageclass.yaml
```
4.Create Persistent Volume Claim using above Storage Class:

```yaml
# jenkins-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: jenkins-azurefile
```
```bash
kubectl apply -f jenkins-pvc.yaml
```
5.Create custom docker image with Jenkins and install plugins.
Due to problems, I encountered installing Jenkins with default settings I had to create custom image. Customization of Jenkins is a complex task and each part needs a bit of tweaking.
[Link to custom image](https://hub.docker.com/repository/docker/adamkielar/jenkins-runner/general).

```Dockerfile
# Dockerfile.jenkins
FROM jenkins/jenkins:2.375.2
USER root
RUN apt-get update && apt-get install -y lsb-release vim && \
    curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg && \
    echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && apt-get install -y docker-ce-cli
COPY ./plugins.txt .
USER jenkins
RUN jenkins-plugin-cli --plugins -f plugins.txt
```
```
#plugins.txt
kubernetes:1.31.3
workflow-job:1189.va_d37a_e9e4eda_
workflow-aggregator:581.v0c46fa_697ffd
git:5.0.0
git-client:4.0.0
github-branch-source:1696.v3a_7603564d04
configuration-as-code:1569.vb_72405b_80249
kubernetes-credentials-provider:0.22
job-dsl:1.81
credentials:1214.v1de940103927
```
6.Customize values.yaml file for Helm install:

[Here you can find a template for that file](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/values.yaml).

<blockquote class="prompt-info">
I will not post my values.yaml file here since it has 427 lines. <a href="https://github.com/adamkielar/jenkins-runner/blob/master/jenkins-values.yaml" target="_blank">You can check it here</a>.
</blockquote>

We will focus only on specific options:
- resource requests and limits for pod
- prometheus
- PVC
- storageClass
- serviceAccount
- plugins (we will comment them out)
- backup (we will turn it off for demo)

```bash
helm install jenkins jenkinsci/jenkins --namespace jenkins -f jenkins-values.yaml
```
7.Confirm that pod is running. It may take upto 10 minutes to start with some container restarts along the way.
```bash
kubectl -n jenkins get pods
```

## Configure Jenkins

1. Get password for admin panel. Login: `admin`
```bash
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```
2. Connect to Jenkins and log in:
```bash
# Open new tab in terminal
kubectl -n jenkins port-forward jenkins-0 8080:8080
```
```bash
open http://localhost:8080/login
```
3. Update plugins. **Manage Jenkins > Manage Plugins > Updates**
4. Enable JGIT plugin. **Manage Jenkins > Global Tool Configuration > Git , change to JGit**
<blockquote class="prompt-info">
The reason we change to jgit is that default git plugin has problem with writing temporary credentials with proper permissions.
This is solution to this error:
</blockquote>
```bash
stdout: 
stderr: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0555 for '/var/jenkins_home/caches/git-0aa16db65c903d3ced737f801b217112@tmp/jenkins-gitclient-ssh2425211278542515051.key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "/var/jenkins_home/caches/git-0aa16db65c903d3ced737f801b217112@tmp/jenkins-gitclient-ssh2425211278542515051.key": bad permissions
Permission denied (publickey).
fatal: Could not read from remote repository.
```

### Set up connection with GitHub for private repository

1. Generate SSH key and save it to a file.
```bash
ssh-keygen -t ed25519
```
2. Copy the public key and add it Github repository
```bash
cat 'path to your file id_ed25519.pub'
```
![GitHub deploy key](/assets/post5/deploy-key.png)
3. Create Kubernetes Secret with private key:
```yaml
# github-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-github-ssh
  namespace: jenkins
  labels:
    "jenkins.io/credentials-type": "basicSSHUserPrivateKey"
  annotations:
    "jenkins.io/credentials-description" : "ssh github.com:adamkielar/jenkins-runner"
stringData:
  privateKey: | # Add private key. `cat 'path to your file id_ed25519``
    -----BEGIN OPENSSH PRIVATE KEY-----
    -----END OPENSSH PRIVATE KEY-----
  username: # Add github username
```
```bash
kubectl apply -f github-secret.yaml
```
4. Add github.com public key to know hosts in **Manage Jenkins > Configure Global Security > Git Host Key Verification Configuration**
```bash
ssh-keyscan github.com
```

### Create New Job
We will create it using UI but we can also create it in configuration file.

1. Create project
![Jenkins 1](/assets/post5/jenkins-1.png)

2. Connect to Github
![Jenkins 2](/assets/post5/jenkins-2.png)

### Create Jenkinsfile with pipeline definition
Update file with ACR name and push changes to your repository.

```yaml
pipeline {
  agent {
    kubernetes {
      defaultContainer 'kaniko'
      yaml '''
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.9.0-debug
    imagePullPolicy: Always
    command:
    - sleep
    args:
    - 99d
    envFrom:
      - secretRef:
          name: acr-sp
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker/
  volumes:
    - name: docker-config
      configMap:
        name: docker-config

'''
    }
  }
  stages {
    stage('Kaniko Build and Push') {
      steps {
        sh '/kaniko/executor --dockerfile=${PWD}/Dockerfile -c ${PWD} --cache=true --destination=jenkinsacr9482.azurecr.io/jenkins-demo:v1.0.0'
      }
    }
    stage('Deploy Application') {
      agent {
    kubernetes {
      defaultContainer 'kubectl'
      yaml '''
  kind: Pod
  spec:
    containers:
    - name: kubectl
      image: quay.io/tfgco/kubectl
      imagePullPolicy: Always
      command:
      - sleep
      args:
      - 99d
'''
    }
  }
      steps {
        sh 'kubectl apply -f /home/jenkins/agent/workspace/jenkins-on-aks_master/pod.yaml'
      }
    }
  }
}
```


### Build pipeline

1. Scan repository to discover Jenkinsfile

![Jenkins 3](/assets/post5/jenkins-3.png)
2. Build pipeline and check output
![Jenkins 4](/assets/post5/jenkins-4.png)
3. Check Grafana
![Grafana 2](/assets/post5/grafana-2.png)
4. Check ACR
![ACR](/assets/post5/acr-1.png)
5. Check if pipeline finished with success
![Jenkins 5](/assets/post5/jenkins-5.png)
6. Confirm that application pod is running
![App 1](/assets/post5/app-1.png)

## Video walkthrough

<iframe width="560" height="315" src="https://www.youtube.com/embed/RVeEQTr0Cqo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Known errors

Besides git plugin error that I describe above, I had also problems with:

1. Problem with log in to Jenkins due to csrf:

In container running Jenkins change find this lines in `/var/jenkins_home/config.xml` and change them as you can see below.
```xml
  <useSecurity>false</useSecurity> 
  <!--<authorizationStrategy class="hudson.security.AuthorizationStrategy$Unsecured"/>
  <securityRealm class="hudson.security.SecurityRealm$None"/>-->
```

2. If you have problem installing Jenkins due to plugins errors, first install clean instance and then add plugins.
3. If you have problems with mounting volumes to container, check if you mount volume with proper uid and gid set.

