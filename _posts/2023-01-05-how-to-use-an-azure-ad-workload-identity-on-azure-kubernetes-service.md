---
layout: post
title: "How to use an Azure AD workload identity on Azure Kubernetes Service?"
tags: ["azure", "aks", "fastapi", "workload identity"]
---

**Applications that we deploy in AKS clusters require AAD application credentials or managed identity to access AAD-protected resources.
Azure AD Workload Identity for Kubernetes integrates with the capabilities native to Kubernetes to federate with external identity providers.**

![Workload diagrma](/assets/post3/workload-diagram-1.png)

## Overview

In this post we will:
1. Create an AKS cluster.
2. Create Azure Key Vault and Azure Storage Account.
3. Set up Azure AD workload identity.
4. Deploy [FastApi](https://fastapi.tiangolo.com/) application.

Our application will perform the following tasks:

* get a secret from Key Vault
* call [OpenAI](https://openai.com/api/) API
* save response to blob in Azure Storage

![Infra diagram](/assets/post3/app-diagram-1.png)

## Prerequisits
We need to Register the `EnableWorkloadIdentityPreview` feature flag.
Follow [documentation](https://learn.microsoft.com/en-us/azure/aks/learn/tutorial-kubernetes-workload-identity#register-the-enableworkloadidentitypreview-feature-flag){:target="_blank"} to enable it.

## Create resources

### Create environment variables
We will set up variables for our convenience:

```bash
resourceGroup="workload-identity-rg"
aksIdentity="aks-identity"
aksName="aks-workload-identity"
kvName="aks-$RANDOM-kv"
storageName="aksstorage$RANDOM"
workloadIdentity="workload-identity"
```

### Create a resource group

```bash
az group create --name $resourceGroup --location eastus
```

### Create a user-assigned managed identity for Kubernetes control plane

```bash
appId=$(az identity create --name $aksIdentity --resource-group $resourceGroup --query id --output tsv)
```

### Create Admin Group in Azure Active Directory for AKS

```bash
groupId=$(az ad group create --display-name AKSADMINS --mail-nickname AKSADMINS --query id -o tsv)
```

### Add user to AKSADMINS group

```bash
userId=$(az ad user show --id <add user id, for me, it is my email> --query id -o tsv)
az ad group member add --group $groupId --member-id $userId
```

### Create Azure Kubernetes Service cluster

```bash
az aks create \
--resource-group $resourceGroup \
--name $aksName \
--location eastus \
--assign-identity $appId \
--enable-managed-identity \
--enable-oidc-issuer \
--enable-workload-identity \
--enable-aad \
--aad-admin-group-object-ids $groupId \
--node-count 1 \
--node-vm-size "Standard_B2s"
```

We can find a description of each flag in [documentation](https://learn.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create){:target="_blank"}.

### Get AKS credentials

```bash
az aks get-credentials --resource-group $resourceGroup --name $aksName
```

### Get the OIDC Issuer URL

```bash
oidcUrl="$(az aks show --name $aksName --resource-group $resourceGroup --query "oidcIssuerProfile.issuerUrl" -o tsv)"
```

### Create Key Vault

```bash
az keyvault create --name $kvName \
--resource-group $resourceGroup
```

### Add OpenAI API key to Key Vault secrets

```bash
az keyvault secret set --vault-name $kvName \
--name "CHATGPT-API-KEY" \
--value <your api key value>
```

![OpenAI API Keys](/assets/post3/openapi-keys.png)

### Create Storage Account

```bash
az storage account create --name $storageName \
--resource-group $resourceGroup \
--allow-blob-public-access false \
--kind StorageV2 \
--public-network-access "Enabled" \
--sku "Standard_LRS" \
--encryption-services blob
```

### Add Storage Container

```bash
az storage container create \
--account-name  $storageName \ 
--name "chatgpt"
```

### Create a user-assigned managed identity for workflow identity

```bash
az identity create --name $workloadIdentity --resource-group $resourceGroup
```

### Grant permission to access the secret in Key Vault

```bash
clientId=$(az identity show --name $workloadIdentity --resource-group $resourceGroup --query clientId -o tsv)

az keyvault set-policy --name $kvName \
--secret-permissions get \
--spn $clientId
```

### Grant permission to access Storage Account

```bash
storageId=$(az storage account show --name $storageName --resource-group $resourceGroup --query id -o tsv)
az role assignment create --assignee $clientId --role 'Storage Blob Data Contributor' --scope $storageId
```

### Create Kubernetes service account

Create a `sa.yaml` file and insert a clientId of managed identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: <insert clientId>
  labels:
    azure.workload.identity/use: "true"
  name: workload-sa
  namespace: default
```

Apply file.

```bash
kubectl apply -f sa.yaml
```

### Create federated identity credentials

```bash
az identity federated-credential create \
--name "aks-federated-credential" \
--identity-name $workloadIdentity \
--resource-group $resourceGroup \
--issuer "${oidcUrl}" \
--subject "system:serviceaccount:default:workload-sa"
```

## Deploy an application

We will use the following docker image:
* [adamkielar/workload-identity-demo:v1.0.0](https://hub.docker.com/repository/docker/adamkielar/workload-identity-demo)

### Application code

Let's look at the most important parts of code

```python

```

### Get Key Vault URL

```bash
kvUrl=$(az keyvault show --name $kvName --resource-group $resourceGroup --query properties.vaultUri -o tsv)
```

### Create Kubernetes secret

```bash
kubectl -n default create secret generic workload-demo-secrets --from-literal=KV-URL=$kvUrl --from-literal=ACCOUNT-NAME=$storageName
```

### Create Kubernetes pod

Create a `pod.yaml` file.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workflow-demo-pod
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-sa
  containers:
    - image: docker.io/adamkielar/workload-identity-demo:v1.0.0
      name: workflow-demo-container
      envFrom:
      - secretRef:
          name: workload-demo-secrets
  nodeSelector:
    kubernetes.io/os: linux
```

Apply file

```bash
kubectl apply -f pod.yaml
```

Confirm that pod is running

```bash
kubectl describe pod workload-demo-pod
```

### Test an application

Forward ports

```bash
kubectl port-forward workload-demo-pod 8000:8000
```

Curl endpoint.

```bash
curl -X POST http://localhost:8000/api/chatgpt -H 'Content-Type: application/json' -d '{"prompt": "Azure workload identity"}'
```

Check pod logs.

```bash
kubectl logs --tail=20 workload-demo-pod
```

Check Storage Account.
<image>

## Video walkthrough