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
1. We need to Register the `EnableWorkloadIdentityPreview` feature flag.
Follow [documentation](https://learn.microsoft.com/en-us/azure/aks/learn/tutorial-kubernetes-workload-identity#register-the-enableworkloadidentitypreview-feature-flag){:target="_blank"} to enable it.
2. [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/#kubectl)

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
az role assignment create \
--assignee $clientId \
--role 'Storage Blob Data Contributor' \
--scope $storageId
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
# Imports
import json
import logging
import os
import secrets
from dataclasses import dataclass

from aiohttp import ClientSession
from azure.keyvault.secrets.aio import SecretClient
from azure.keyvault.secrets._models import KeyVaultSecret
from azure.identity.aio import DefaultAzureCredential
from azure.storage.blob.aio import BlobServiceClient
from fastapi import APIRouter
from fastapi import status
from pydantic import BaseModel

# FastApi router
router = APIRouter()


# Pydantic model
class Question(BaseModel):
    prompt: str


# Retrieve secret `CHATGPT-API-KEY` from Key Vault
@dataclass
class KvSecretHandler:
    kv_url: str = os.getenv("KV-URL", "")
    secret_name: str = "CHATGPT-API-KEY"

    async def get_secret(self) -> KeyVaultSecret:
        default_credential = DefaultAzureCredential()
        kv_client = SecretClient(
            vault_url=self.kv_url,
            credential=default_credential
            )

        async with kv_client:
            async with default_credential:
                return await kv_client.get_secret(self.secret_name)


# Save response from ChatGPT to txt file 
# and upload it to `chatgpt` container in Storage Account.
@dataclass
class StorageBlobHandler:
    account_url = f"https://{os.getenv('ACCOUNT-NAME', '')}.blob.core.windows.net"

    async def upload_blob(self, blob_answer: bytes) -> None:
        file_name = secrets.token_urlsafe(5)
        default_credential = DefaultAzureCredential()
        blob_service_client = BlobServiceClient(
            account_url=self.account_url,
            credential=default_credential
            )

        async with blob_service_client:
            async with default_credential:
                container_client = blob_service_client.get_container_client(
                    container="chatgpt"
                    )

                blob_client = container_client.get_blob_client(
                    blob=f"{file_name}.txt"
                    )

                await blob_client.upload_blob(data=blob_answer)


# Call OpenAI API with question and return response
@dataclass
class ChatGptApiCallHandler:
    async def process_question(self, chatgpt_key: str, prompt: str) -> bytes:
        headers = {
            "Authorization": f"Bearer {chatgpt_key}",
            "Content-Type": "application/json"
        }
        payload = {
            "model": "text-davinci-003",
            "prompt": prompt,
            "max_tokens": 100,
            "temperature": 0
        }
        async with ClientSession(headers=headers) as session:
            async with session.post(
                url="https://api.openai.com/v1/completions",
                data=json.dumps(payload)
            ) as response:
                return await response.read()


# Application endpoint to POST question, http://localhost:8000/api/chatgpt
@router.post("/chatgpt", status_code=status.HTTP_201_CREATED)
async def send_question(question: Question) -> None:
    chatgpt_key = await KvSecretHandler().get_secret()
    chatgpt_response = await ChatGptApiCallHandler().process_question(
        chatgpt_key.value,question.prompt
        )
    await StorageBlobHandler().upload_blob(chatgpt_response)

```

### Get Key Vault URL

```bash
kvUrl=$(az keyvault show --name $kvName --resource-group $resourceGroup --query properties.vaultUri -o tsv)
```

### Create Kubernetes secret

```bash
kubectl -n default create secret generic workload-demo-secrets \
--from-literal=KV-URL=$kvUrl \
--from-literal=ACCOUNT-NAME=$storageName
```

### Create Kubernetes pod

Create a `pod.yaml` file.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-demo-pod
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-sa
  containers:
    - image: docker.io/adamkielar/workload-identity-demo:v1.0.0
      name: workload-demo-container
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
curl -X POST http://localhost:8000/api/chatgpt \
-H 'Content-Type: application/json' \
-d '{"prompt": "Azure workload identity"}'
```

Check pod logs.

```bash
kubectl logs --tail=20 workload-demo-pod
```

Check Storage Account.
<image>

## Video walkthrough