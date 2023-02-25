---
layout: post
title: "Run self-hosted CI/CD agents on Azure Kubernetes Service - Part 4 - Azure DevOps"
description: "Learn how to run self-hosted CI/CD agents on Azure Kubernetes Service with Azure DevOps in this comprehensive guide. Part 4 covers how to set up and configure agents in Azure DevOps for use in AKS. Improve your development process with this powerful tool!"
tags: ["azure", "aks", "azure devops", "kubernetes"]
---

<img src="/assets/post8/azure-devops-runner.png" width="300">

**Are you looking for a way to run your [Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/user-guide/what-is-azure-devops?view=azure-devops){:target="_blank"} builds and deployments on your own infrastructure? If so, you'll want to check out Azure DevOps self-hosted runners. Self-hosted runners are agents that allow you to run your build and deployment jobs on machines that you control, giving you more flexibility and control over your environment. In this blog post, we'll take a closer look at what Azure DevOps self-hosted runners are, why you might want to use them, and how to set them up for your projects. Whether you're looking to save costs, ensure security and isolation, or build on specialized hardware or software configurations, self-hosted runners can help you improve your Azure DevOps workflow.**

**This post is a continuation of our journey with self-hosted CI/CD agents. I encourage you to check [part 1](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-1-circleci-buildah/){:target="_blank"}, [part 2](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-2-jenkins-kaniko/){:target="_blank"} and [part 3](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-3-github-actions/){:target="_blank"} if you want to see a different approach to that topic.**

## Overview

We will create and configure the following resources:

* AKS cluster with workload identity and kubelet identity.
* Azure Container Registry.
* Role assignment for kubelet identity and workload identity.
* Kubernetes deployment object for self-hosted runner.
* Azure Devops pipeline.

<blockquote class="prompt-info">
<a href="https://github.com/adamkielar/azure-devops-on-aks" target="_blank" rel="noopener noreferrer">Here is a link to GitHub repo with all files for reference</a>.
</blockquote>

## Video walkthrough

<iframe width="560" height="315" src="https://www.youtube.com/embed/2laCzIoDGEA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Create AKS cluster and ACR

1. Run script.
```bash
# After running above script, if there were no errors, variables should be available in terminal.
chmod +x aks.sh
./aks.sh 'add user id, for me, it is my email of AAD user'
```
2. Get credentials to AKS, oidcUrl and test connection.
```bash
az aks get-credentials --resource-group $resourceGroup --name $aksName
export oidcUrl="$(az aks show --name $aksName \
--resource-group $resourceGroup \
--query "oidcIssuerProfile.issuerUrl" -o tsv)"
kubectl get nodes
```

## Set up workload identity

1. Create workload identity.
```bash
workloadIdentity="workload-identity"
workloadClientId=$(az identity create --name $workloadIdentity \
--resource-group $resourceGroup --query clientId -o tsv)
workloadPrincipalId=$(az identity show --name $workloadIdentity \
--resource-group $resourceGroup --query principalId -o tsv)
```
2. Create Kubernetes Service Account in `devops` namespace.
```yaml
# devops-runner-sa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops
  labels:
    name: devops
---
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: <insert workloadClientId> # echo $workloadClientId
  labels:
    azure.workload.identity/use: "true"
  name: workload-sa
  namespace: devops
```
```bash
kubectl apply -f devops-runner-sa.yaml
```
3. Create ClusterRole and ClusterRoleBinding for Service Account. Update workloadPrincipalId in file. 
```yaml
# devops-roles.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: devops
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
  - endpoints
  - deployments
  - deployments/scale
  - daemonsets
  - configmaps
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: devops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: devops
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:devops
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: # echo $workloadPrincipalId
```
```bash
kubectl apply -f devops-roles.yaml
```
4. Create federated identity credentials.
```bash
az identity federated-credential create \
--name "aks-federated-credential" \
--identity-name $workloadIdentity \
--resource-group $resourceGroup \
--issuer "${oidcUrl}" \
--subject "system:serviceaccount:devops:workload-sa"
```
5. Create custom role for workload identity.
Create acrbuild.json file with following definition. Replace `{YOUR SUBSCRIPTION}` with your subscription id.
```json
{
  "Name": "AcrBuild",
  "IsCustom": true,
  "Description": "Can read, push, pull and list builds.",
  "Actions": [
    "Microsoft.ContainerRegistry/registries/read",
    "Microsoft.ContainerRegistry/registries/pull/read",
    "Microsoft.ContainerRegistry/registries/push/write",
    "Microsoft.ContainerRegistry/registries/scheduleRun/action",
    "Microsoft.ContainerRegistry/registries/runs/*",
    "Microsoft.ContainerRegistry/registries/listBuildSourceUploadUrl/action"
  ],
  "AssignableScopes": [
    "/subscriptions/{YOUR SUBSCRIPTION}"
  ]
}
```
```bash
az role definition create --role-definition acrbuild.json
```
6. Assign AcrBuild role to workload identity.
```bash
az role assignment create --assignee $workloadClientId \
--role 'AcrBuild' --scope $acrId
```
7. Assign Azure Kubernetes Service Cluster User Role and Azure Kubernetes Service RBAC Writer to workload identity.
```bash
az role assignment create \
--role "Azure Kubernetes Service Cluster User Role" \
--assignee $workloadPrincipalId \
--scope $aksId
```
```bash
az role assignment create \
--role "Azure Kubernetes Service RBAC Writer" \
--assignee $workloadPrincipalId \
--scope "$aksId/namespaces/devops"
```

## Install Azure Devops self-hosted agent

### Personal access token (PAT) and Agent Pool

As a first step, we must register the runner. We need to have permissions to administer the agent queue if we want to complete that step.

1. Sign in to Azure DevOps organization: `https://dev.azure.com/{your_organization}`.
2. Create personal access token.
![Azure DevOps user settings](/assets/post8/pat-1.png)
![Create PAT token](/assets/post8/pat-2.png)
3. Create an Agent pool
![Azure DevOps Organization settings](/assets/post8/agent-1.png)

### Create an image for Azure DevOps runner
We will build custom image useing [new version](https://github.com/microsoft/azure-pipelines-agent/releases) of the runner, pre-release v3.217.1 . We will also add az cli, kubectl. 

```Dockerfile
FROM ubuntu:22.04
USER root
RUN apt-get -y update && apt-get install -y curl && \
    curl -sL https://aka.ms/InstallAzureCLIDeb | bash && az aks install-cli && \
    curl -fsSL https://get.docker.com -o get-docker.sh && sh ./get-docker.sh && \
    mkdir devops-runner && cd devops-runner && \
    curl -o vsts-agent-linux-x64-3.217.1.tar.gz -L https://vstsagentpackage.azureedge.net/agent/3.217.1/vsts-agent-linux-x64-3.217.1.tar.gz && \
    tar xzf ./vsts-agent-linux-x64-3.217.1.tar.gz && \
    apt-get clean

RUN addgroup --gid 110 devops && adduser devops --uid 111 --system && adduser devops devops && \
  chown -R devops:devops devops-runner

USER devops
```
```bash
az acr build -f Dockerfile.runner -t devops-runner:v1.0.0 -r $acrName -g $resourceGroup .
```

### Create Storage Class and Persistent Volume Claim
```yaml
# devops-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: devops-azurefile
provisioner: file.csi.azure.com
mountOptions:
  - uid=110
  - gid=111
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
parameters:
  skuName: Standard_LRS
```

```yaml
# devops-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: devops-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: devops-azurefile
```
```bash
kubectl apply -f devops-storageclass.yaml
kubectl apply -f devops-pvc.yaml
```

### Create Deployment

```yaml
# devops-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-deployment
  namespace: devops
  labels:
    app: devops-runner
    azure.workload.identity/use: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devops-runner
      azure.workload.identity/use: "true"
  template:
    metadata:
      labels:
        app: devops-runner
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: workload-sa
      containers:
      - name: devops-runner
        image: devopsacr14044.azurecr.io/devops-runner:v1.0.0 # Add your ACR respository
        command:
        - sleep
        args:
        - 99d
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        volumeMounts:
          - mountPath: "/home/devops"
            name: runner-data
      volumes:
        - name: runner-data
          persistentVolumeClaim:
            claimName: devops-pvc
```
```bash
kubectl apply -f devops-deployment.yaml
```

### Set up runner inside pod

1. Check if pod is running and connect to container.

```bash
kubectl -n devops get pods
kubectl -n devops exec -it <POD NAME> -- sh
```
2. Configure runner
```bash
./devops-runner/config.sh
```
![Configure Azure Devops runner](/assets/post8/agent-3.png)

We need to provide additional information here.
  * Azure DevOps URL eg., https://dev.azure.com/org-name
  * PAT token
  * Name of the agent pool registered in Azure DevOps Services. Defaults to 'Default' pool
3. Start runner
```bash
./devops-runner/run.sh
```
We can always automate this process later.
![Connect runner to Azure Devops](/assets/post8/agent-4.png)

## Create Azure Devops pipeline

1. In `.ci` folder create `azure-pipelines.yml`.

Update name of ACR registry and optionally AKS name and resource group.
```yaml
name: $(Date:yyyyMMdd)$(Rev:.r)

pool: DevOpsOnAks

trigger:
  batch: true

stages:
  - stage: buildPushDeploy
    displayName: Build, Push and Deploy
    jobs:
      - job: build
        steps:
          - checkout: self
            clean: true

          - script: |
              set -euo pipefail
              az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID --federated-token $(cat $AZURE_FEDERATED_TOKEN_FILE)
              az acr build -f $(Agent.BuildDirectory)/s/Dockerfile -t devopsacr14044.azurecr.io/devops-demo:$(Build.SourceVersion) -r devopsacr14044 .
            displayName: Log in and push image to ACR

          - script: |
              set -euo pipefail
              az aks get-credentials --resource-group devops-runner-rg --name aks-devops-runner --overwrite-existing
              kubelogin convert-kubeconfig -l workloadidentity
              sed 's|IMAGE|devopsacr14044.azurecr.io/devops-demo|g; s/TAG/$(Build.SourceVersion)/g' $(Agent.BuildDirectory)/s/app.yaml | kubectl apply -f -
            displayName: Get AKS credentials and deploy app

```
2. Push changes to GitHub and check status of each resource in Azure Devops.

![Job output](/assets/post8/agent-5.png)

![Azure Devops output 1](/assets/post8/agent-8.png)

![Azure Devops output 2](/assets/post8/agent-6.png)

![Kubectl output](/assets/post8/agent-7.png)

**We successfully managed to deploy our application.**