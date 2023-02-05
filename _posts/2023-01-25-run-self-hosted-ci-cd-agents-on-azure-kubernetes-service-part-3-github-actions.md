---
layout: post
title: "Run self-hosted CI/CD agents on Azure Kubernetes Service - Part 3 - GitHub Actions"
description: "In Part 3 of this series, discover how to run self-hosted CI/CD agents on Azure Kubernetes Service using GitHub Actions. Learn how to set up and configure GitHub Actions for building and deploying your applications on AKS, and take advantage of the power of GitHub Actions for CI/CD automation."
tags: ["azure", "aks", "github", "kubernetes"]
---

<img src="/assets/post6/github-diagram.png" width="300">

**This post is a continuation of our journey with self-hosted CI/CD agents. I encourage you to check [part 1](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-1-circleci-buildah/){:target="_blank"} and [part 2](https://www.adamkielar.pl/posts/run-self-hosted-ci-cd-agents-on-azure-kubernetes-service-part-2-jenkins-kaniko/){:target="_blank"} if you want to see a different approach to that topic. In this post, we will focus on [GitHub Actions](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners){:target="_blank"}.**

## GitHub-hosted vs self-hosted runners

The documentation says that GitHub-hosted runners offer a simple way to run workflows, while self-hosted runners are a great solution if you want to create a more configurable set up in your own environment.

GitHub-hosted runners:

* Receive automatic updates for the operating system and tools.
* Are managed and maintained by GitHub.
* Provide a clean instance for every job execution.
* Use free minutes on your GitHub plan, with per-minute rates applied after surpassing the free minutes.

Self-hosted runners:

* Receive automatic updates for the self-hosted runner application only. 
* You are responsible for updating the operating system and all other software.
* Can use cloud services or local machines that you already pay for.
* Are customizable to your hardware, operating system, software, and security requirements.
* Don't need to have a clean instance for every job execution.
* Are free to use with GitHub Actions, but you are responsible for the cost of maintaining your runner machines.

## Overview

We will create and configure the following resources:

* AKS cluster with workload identity and kubelet identity.
* Azure Container Registry.
* Role assignment for kubelet identity and workload identity.
* Kubernetes objects for workload identity and GitHub runner.

We will also check how to se up a runner with [actions runner controller](https://github.com/actions/actions-runner-controller/blob/master/docs/quickstart.md){:target="_blank"}.

<blockquote class="prompt-info">
<a href="https://github.com/adamkielar/github-runner" target="_blank" rel="noopener noreferrer">Here is a link to GitHub repo with all files for reference</a>.
</blockquote>

## Video walkthrough

<iframe width="560" height="315" src="https://www.youtube.com/embed/1tugWxg_1_0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

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
2. Create Kubernetes Service Account in GitHub namespace.
```yaml
# github-runner-sa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: github
  labels:
    name: github
---
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: <insert workloadClientId> # echo $workloadClientId
  labels:
    azure.workload.identity/use: "true"
  name: workload-sa
  namespace: github
```
```bash
kubectl apply -f github-runner-sa.yaml
```
3. Create ClusterRole and ClusterRoleBinding for Service Account. Update workloadPrincipalId in file. 
```yaml
# github-roles.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: github
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
  name: github
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: github
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:github
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: # echo $workloadPrincipalId
```
```bash
kubectl apply -f github-roles.yaml
```
4. Create federated identity credentials.
```bash
az identity federated-credential create \
--name "aks-federated-credential" \
--identity-name $workloadIdentity \
--resource-group $resourceGroup \
--issuer "${oidcUrl}" \
--subject "system:serviceaccount:github:workload-sa"
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
--scope "$aksId/namespaces/github"
```

## Install GitHub Actions self-hosted runner

![GitHub create runner](/assets/post6/github-1.png)

### Create docker image for GitHub runner.
We will use information from above instructions. We will add az cli, kubectl. Size of image is far from perfect. Build and push it to our ACR.

```Dockerfile
FROM ubuntu:22.04
USER root
RUN apt-get -y update && apt-get install -y curl && \
    curl -sL https://aka.ms/InstallAzureCLIDeb | bash && az aks install-cli && \
    curl -fsSL https://get.docker.com -o get-docker.sh && sh ./get-docker.sh && \
    mkdir actions-runner && cd actions-runner && \
    curl -o actions-runner-linux-x64-2.301.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.301.1/actions-runner-linux-x64-2.301.1.tar.gz && \
    tar xzf ./actions-runner-linux-x64-2.301.1.tar.gz && ./bin/installdependencies.sh && \
  apt-get clean

RUN addgroup --gid 106 github && adduser github --uid 105 --system && adduser github github && \
  chown -R github:github actions-runner

USER github

EXPOSE 8080
```
```bash
az acr build -f Dockerfile.runner -t github-runner:v1.0.0 -r $acrName -g $resourceGroup .
```
### Create Storage Class and Persistent Volume Claim
```yaml
# github-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: github-azurefile
provisioner: file.csi.azure.com
mountOptions:
  - uid=105
  - gid=106
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
parameters:
  skuName: Standard_LRS
```

```yaml
# github-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: github-pvc
  namespace: github
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: github-azurefile
```
```bash
kubectl apply -f github-storageclass.yaml
kubectl apply -f github-pvc.yaml
```
### Create StatefulSet and Service for runner

```yaml
apiVersion: v1
kind: Service
metadata:
  name: github-runner
  namespace: github
  labels:
    app: github-runner
spec:
  ports:
  - port: 8080
    name: github-runner-port
  clusterIP: None
  selector:
    app: github-runner
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: github-runner
  namespace: github
spec:
  replicas: 1
  minReadySeconds: 10
  serviceName: github-runner
  selector:
    matchLabels:
      app: github-runner
      azure.workload.identity/use: "true"
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
  template:
    metadata:
      labels:
        app: github-runner
        azure.workload.identity/use: "true"
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: workload-sa
      containers:
      - image: githubacr14149.azurecr.io/github-runner:v1.0.0 # Add your ACR respository
        name: github-runner
        imagePullPolicy: Always
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
          - mountPath: "/home/github"
            name: runner-data
      volumes:
        - name: runner-data
          persistentVolumeClaim:
            claimName: github-pvc
```
```bash
kubectl apply -f statefulset.yaml
```

### Set up runner inside pod

1. Check if pod is running and connect to container.

```bash
kubectl -n github get pods
kubectl -n github exec -it github-runner-0 -- sh
```
2. Configure runner
```bash
./actions-runner/config.sh --url https://github.com/adamkielar/github-runner --token <YOU TOKEN>
```
![Github runner 2](/assets/post6/github-2.png)
3. Start runner
```bash
./actions-runner/run.sh
```
![Github runner 3](/assets/post6/github-3.png)

## Create GitHub workflow

1. In `.github/workflows` create `deploy-app.yaml`.

Update name of ACR registry and optionally AKS name and resource group.
```yaml
# deploy-app.yaml
name: Build image, push to ACR, deploy to AKS
concurrency: 
  group: deployment

on:
  push:
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy application
    runs-on: self-hosted
    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      
      - name: Login with workload identity
        run: az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID --federated-token $(cat $AZURE_FEDERATED_TOKEN_FILE)

      - name: Build image with AZ ACR
        run: |
          az acr build -f /actions-runner/_work/github-runner/github-runner/Dockerfile -t githubacr14149.azurecr.io/github-demo:${{ github.sha }} -r githubacr14149 .
      
      - name: Get AKS credentials
        run: |
          az aks get-credentials --resource-group github-runner-rg --name aks-github-runner --overwrite-existing
          kubelogin convert-kubeconfig -l workloadidentity

      - name: Deploy application
        run: |
          sed 's|IMAGE|githubacr14149.azurecr.io/github-demo|g; s/TAG/${{ github.sha }}/g' /actions-runner/_work/github-runner/github-runner/pod.yaml | kubectl apply -f -

```

2. Push changes to GitHub and check status of each resource.

![Job output](/assets/post6/github-4.png)

![GitHub output](/assets/post6/github-5.png)

![Kubectl output](/assets/post6/github-6.png)

## Set up self-hoster runner with Actions Runner Controller

Let's follow the [documentation](https://github.com/actions/actions-runner-controller/blob/master/docs/quickstart.md){:target="_blank"}
and see if the steps are straightforward.

1. Install cert-manager.
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```
2. Generate a Personal Access Token in GitHub for your repo.
- repo (all)
- admin:public_key - read:public_key
- admin:repo_hook - read:repo_hook
- admin:org_hook
- notifications
- workflow
3. Deploy controller using helm.
```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
```
```bash
helm upgrade --install --namespace actions-runner-system --create-namespace\
  --set=authSecret.create=true\
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE"\
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```
4. Create runnerdeployment.yaml.
```yaml
# runnerdeployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: github-runnerdeploy
  namespace: github
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-runner-v2
  template:
    metadata:
      labels:
        app: github-runner-v2
        azure.workload.identity/use: "true"
    spec:
      repository: adamkielar/github-runner # change to your repository
      labels:
        - github-runner
      ephemeral: true
      serviceAccountName: workload-sa
```
```bash
kubectl apply -f runnerdeployment.yaml 
```
5. Confirm pods are running.
![Controller pods](/assets/post6/github-7.png)
6. Create `.github/workflow/deploy-app-v2.yaml` file. We will modify our pipeline a bit. We will add [install_libs](https://github.com/adamkielar/github-runner/blob/main/install_libs.sh) script.

```yaml
name: Build image, push to ACR, deploy to AKS
concurrency: 
  group: deployment

on:
  push:
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy application
    runs-on: github-runner
    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      
      - name: Install Azure CLI, kubectl
        run: chmod +x install_libs.sh && ./install_libs.sh

      - name: Install kubelogin
        run: sudo az aks install-cli
      
      - name: Login with workload identity
        run: az login --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID --federated-token $(cat $AZURE_FEDERATED_TOKEN_FILE)

      - name: Build image with AZ ACR
        run: |
          az acr build -f /runner/_work/github-runner/github-runner/Dockerfile -t githubacr14149.azurecr.io/github-demo:${{ github.sha }} -r githubacr14149 .
      
      - name: Get AKS credentials
        run: |
          az aks get-credentials --resource-group github-runner-rg --name aks-github-runner --overwrite-existing
          export KUBECONFIG=/home/runner/.kube/config
          kubelogin convert-kubeconfig -l workloadidentity

      - name: Deploy application
        run: |
          sed 's|IMAGE|githubacr14149.azurecr.io/github-demo|g; s/TAG/${{ github.sha }}/g' /runner/_work/github-runner/github-runner/pod.yaml | kubectl apply -f -
```
7. Delete `github-demo-pod`.
```bash
kubectl -n github delete pod github-demo-pod
```
8. Push changes to GitHub and check result.

![GitHub deploy](/assets/post6/github-8.png)


**We successfully managed to deploy our application.**