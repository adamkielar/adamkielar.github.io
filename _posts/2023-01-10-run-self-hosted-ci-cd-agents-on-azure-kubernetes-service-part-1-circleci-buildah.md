---
layout: post
title: "Run self-hosted CI/CD agents on Azure Kubernetes Service - Part 1 - CircleCI + Buildah"
description: "In Part 1 of this series, discover how to run self-hosted CI/CD agents on Azure Kubernetes Service using CircleCI and Buildah. Learn how to set up and configure CircleCI and Buildah to build and deploy your applications on AKS with ease."
tags: ["azure", "aks", "circleci", "kubernetes"]
---

![Diagram](/assets/post4/circle-runner-diagram.png)

**We use self-hosted CI/CD when we want to run and manage our own continuous integration and delivery (CI/CD) pipeline, rather than using a cloud-based service like CircleCI, Azure DevOps, or GitHub Actions. One way to do this is to host the agents on a Kubernetes cluster, which can provide scalability and resource isolation for your build processes. By self-hosting our CI/CD agents on Kubernetes, we have greater control over the infrastructure and can customize the build process to fit our specific needs. However, it also requires more setup and maintenance effort on our part.**

## Overview

We will:

1. Create AKS cluster with workload identity enabled. We touched on that topic in [this post](https://www.adamkielar.pl/posts/how-to-use-an-azure-ad-workload-identity-on-azure-kubernetes-service/){:target="_blank"} so please take the time to read it if you are not familiar with this setup.
2. Create Azure Container Registry to store our images
3. Create Persistent Volume Claim
3. Create CircleCI Agent and simple config

## Video walkthrough

<iframe width="560" height="315" src="https://www.youtube.com/embed/qVTGgiRHFk4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Prerequisites

* Kubernetes 1.12+
* Helm 3.x
* [Token for a resource class](https://circleci.com/docs/runner-faqs/#what-is-a-runner-resource-class){:target="_blank"}
* [jq tool](https://stedolan.github.io/jq/){:target="_blank"} if you want to run all commands
* [Dockerfile](https://github.com/adamkielar/circleci-runner){:target="_blank"}

## Create an AKS cluster with workload identity

[I created script with necessary commands to provision a basic setup](https://github.com/adamkielar/circleci-runner/blob/main/ask.sh){:target="_blank"}. This is the same setup as in [last post](https://www.adamkielar.pl/posts/how-to-use-an-azure-ad-workload-identity-on-azure-kubernetes-service/){:target="_blank"}. Let's run it.

```bash
chmod +x aks.sh
./aks.sh 'add user id, for me, it is my email of AAD user'
```

After running above script, if there were no errors, variables should be available in terminal.
If you will have any problem running it, let me know and I will try to help. I use [zsh](https://www.zsh.org/){:target="_blank"} terminal.

1. Create a user-assigned managed identity for workload identity
```bash
clientId=$(az identity create --name $workloadIdentity --resource-group $resourceGroup --query clientId -o tsv)
```

2. Grant permission to access the secret in Key Vault
```bash
az keyvault set-policy --name $kvName \
--secret-permissions get \
--spn $clientId
```

## Create Azure Container Registry

We will create ACR and we will enable admin login. We will add these credentials to Key Vault.  Later in this post, we will push an image to ACR using [Buildah](https://buildah.io/){:target="_blank"} which does not work currently with Azure Managed Identity setup without Docker.

```bash
acrName='circleciacr'
acrId=$(az acr create --name $acrName --resource-group $resourceGroup --sku Basic --admin-enabled --query id -o tsv)
```

### Get credentials for ACR and add them to Key Vault

```bash
ACRUSER=$(az acr credential show -n $acrName | jq -r .username)
ACRPASSWORD=$(az acr credential show -n $acrName | jq -r '.passwords | .[0].value')
az keyvault secret set --vault-name $kvName \
--name "ACRUSER" \
--value $ACRUSER
az keyvault secret set --vault-name $kvName \
--name "ACRPASSWORD" \
--value $ACRPASSWORD
```

### Create SecretProviderClass

We need this object if we want to access Key Vault from AKS cluster.


<ins>Create file secretprovider.yaml and add missing variables and apply.</ins>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: circleci
  labels:
    name: circleci
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aks-kv-workload-identity
  namespace: circleci
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"          
    clientID: # az identity show --name $workloadIdentity --resource-group $resourceGroup --query clientId -o tsv
    keyvaultName: # echo $kvName
    cloudName: ""
    objects:  |
      array:
        - |
          objectName: ACRUSER
          objectType: secret
          objectVersion: ""
        - |
          objectName: ACRPASSWORD
          objectType: secret
          objectVersion: ""
    tenantId: # az account show --query tenantId -o tsv
```

```bash
kubectl apply -f secretprovider.yaml
```

### Create Service Account, Role and Rolebinding for CircleCI agent

<ins>Create file sa.yaml, add missing variable and apply.</ins>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id:  # az identity show --name $workloadIdentity --resource-group $resourceGroup --query clientId -o tsv
  labels:
    azure.workload.identity/use: "true"
  name: circleci-sa
  namespace: circleci
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: circleci
  name: circleci-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "watch", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["watch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: circleci-rolebinding
  namespace: circleci
subjects:
- kind: ServiceAccount
  name: circleci-sa
  namespace: circleci
roleRef:
  kind: Role
  name: circleci-role
  apiGroup: rbac.authorization.k8s.io
```

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
--subject "system:serviceaccount:circleci:circleci-sa"
```

## Setup CircleCI

**Enable Self-Hosted Runners in CircleCI and create resource class. During that process You will optain access token.
[Follow documentation](https://circleci.com/docs/runner-installation){:target="_blank"}.**

### Create secret object with CircleCI token

```bash
kubectl -n circleci create secret generic circleci-token-secrets --from-literal=circleci-runner.resourceClass='ADD TOKEN'
```

### Create Persistent Volume Claim

We will use storage class that is already created in AKS: `azurefile-csi`.

<ins>Create file pvc.yaml and apply.</ins>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-circleci
  namespace: circleci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: azurefile-csi
```

```bash
kubectl apply -f pvc.yaml
```

### Create CircleCI agent

Install `fuse-overlay` program according to [documentation](https://circleci.com/docs/container-runner/#using-the-buildah-image){:target="_blank"}.

We will create custom values.yaml file.

```yaml
agent:
  customSecret: circleci-token-secrets
  serviceAccount.create: false
  rbac.create: false
  resourceClasses:
   circleci-runner/resourceClass:
    metadata:
    annotations:
      custom.io: circleci-runner
    spec:
      serviceAccountName: circleci-sa
      containers:
        - resources:
            limits:
              github.com/fuse: 1
          volumeMounts:
            - name: agent-store
              mountPath: /home/build/
            - name: secrets-store
              mountPath: "/mnt/secrets-store"
              readOnly: true
          securityContext:
            privileged: true
            runAsUser: 1000
      volumes:
        - name: agent-store
          persistentVolumeClaim:
            claimName: pvc-circleci
        - name: secrets-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "aks-kv-workload-identity"
```

Install agent with Helm.

```bash
helm install container-agent container-agent/container-agent --namespace circleci -f values.yaml
```

Confirm if deployment is created.
![Agent created](/assets/post4/agent-created.png)

### Create config.yaml for CircleCI pipeline

```yaml
version: 2.1

jobs:
  build-push:
    docker:
      - image: quay.io/buildah/stable:v1.28.0
    resource_class: azuretour/aksrunner
    steps:
      - checkout
      - run:
          name: Build and Push
          command: |
            ACRUSER=$(cat /mnt/secrets-store/ACRUSER)
            ACRPASSWORD=$(cat /mnt/secrets-store/ACRPASSWORD)
            buildah bud --format docker --layers -f ./Dockerfile -t circleciacr.azurecr.io/demo-image:v1.0.0 .
            buildah push --creds=${ACRUSER}:${ACRPASSWORD} circleciacr.azurecr.io/demo-image:v1.0.0
workflows:
  build-push-workflow:
    jobs:
      - build-push
```

### Push our changes to the repository and check the job result.

<ins>Confirm that pod for circleci job is created.</ins>

![Job pod 1](/assets/post4/pod1.png)

![Job pod 2](/assets/post4/pod2.png)

<ins>Confirm that CircleCI job is complete.</ins>

![CircleCI runner](/assets/post4/circleci-runner-success.png)

<ins>Confirm that image is in ACR.</ins>

![ACR image](/assets/post4/acr-image.png)

<blockquote class="prompt-info">
According to circleci documentation we can add `runAsNonRoot: true
` to `securityContext`  but during my tests I was unable to mount fuse device plugin without --priviliged flag in AKS with default settings.
We need to be able to modify pod that circle agent creates.
</blockquote>

<blockquote class="prompt-warning">
Error: mount /var/lib/containers/storage/overlay:/var/lib/containers/storage/overlay, flags: 0x1000: permission denied
</blockquote>

Here are great articles from RedHat for further reading:

* [https://www.redhat.com/sysadmin/building-buildah](https://www.redhat.com/sysadmin/building-buildah){:target="_blank"}
* [https://www.redhat.com/sysadmin/podman-inside-kubernetes](https://www.redhat.com/sysadmin/podman-inside-kubernetes){:target="_blank"}

## Conclusion
I would like to have more control over pod which is created by circleci-agent. This solution is not flexible enough for me yet and there are security concerns.
