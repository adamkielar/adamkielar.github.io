---
layout: post
title: "How to use OpenID Connect identity tokens to authenticate CircleCI jobs with Azure"
tags: ["azure", "circleci", "openid"]
---

## Why use OpenID Connect in CI/CD?

* We do not have to store long-lived credentials as secrets in our CI/CD tools.
* We do not have to rotate credentials since they are no longer static.
* We have more granular control over how workflows can use credentials.
* We follow best practices in terms of authentication and authorization.

## Overview

![circleci-oidc-diagram](/assets/post2/circleci-oidc-diagram.png)

When a CircleCI job starts, CircleCI signs an OpenID Connect token and makes it available to a job. 
After that, job can present this token to Azure, which verifies its authenticity, grants a job temporary credentials, and permits it to take defined actions.

## Prerequisites

* Access to Azure Subscription with Owner access level.
* Access to Azure Active Directory Tenant with at least the Application Developer access level. For deployment script we will require Application Administrator
* Azure CLI
* Azure Bicep
* A CircleCI project

## Create resources using Azure CLI

### Create Azure AD application

```bash
appId=$(az ad app create --display-name circleci-oidc --query appId -o tsv)
```

### Create service principal

```bash
az ad sp create --id $appId
```

### Create Azure AD federated identity credentials

We have to make a POST request to MS Graph to add federated credentials. 
Let's check Azure Portal what fields we have to provide to create credentials.

![Azure Portal](/assets/post2/circleci-oidc.png)

* In an Issuer field, we set: `"https://oidc.circleci.com/org/ORGANIZATION_ID"`
* In a Subject identifier field, we set: `"org/ORGANIZATION_ID/project/PROJECT_ID/user/USER_ID"`
* In a Name field, we set: `circleci-federated-identity`
* In a Description field, we set: `CircleCI service account federated identity`
* In an Audience field, we set: `ORGANIZATION_ID`

Now we have to get missing data from CircleCI and Azure.

### Retrieve ORGANIZATION_ID from CircleCI

`ORGANIZATION_ID` is a UUID identifying the current job’s project’s organization. You can find CircleCI organization id by navigating to **Organization Settings > Overview** on the https://app.circleci.com/

### Retrieve PROJECT_ID and USER_ID

`PROJECT_ID` and `USER_ID` are UUIDs that identify the CircleCI project and the user that run the job. We can find PROJECT_ID in **Project Settings > Overview** and USER_ID in **User Settings > Account Integration**

### Retrieve an object id of an Azure AD application

```bash
objectId=$(az ad app show --id $appId --query id -o tsv)
```

### Create a JSON file called body.json where we will store data for an API request

```json
{
  "name": "circleci-federated-identity",
  "issuer": "https://oidc.circleci.com/org/ORGANIZATION_ID",
  "subject": "org/ORGANIZATION_ID/project/PROJECT_ID/user/USER_ID",
  "description": "CircleCI service account federated identity",
  "audiences": [
    "ORGANIZATION_ID"
  ]
}
```


<blockquote class="prompt-info"> Wildcard characters aren't supported in any federated identity credential property value.
In AWS and GCP we can bypass that limitation. I did not find a good solution in Azure yet.
</blockquote>

### Make a MS Graph POST request

```bash
az rest --method POST --uri "https://graph.microsoft.com/beta/applications/$objectId/federatedIdentityCredentials" --body @body.json
```

If we receive a response without errors we can check Azure Portal if a federated identity was created.
![Azure Result](/assets/post2/circleci-identity.png)

### Grant permissions for the service principal

In the below example, we grant the Reader role to our subscription scope.

```bash
az role assignment create --assignee $appId --role Reader --scope /subscriptions/<subscription-id>
```

## Create resources using Azure Bicep

As a prerequisite, we have to use Azure CLI to create an identity and assign a role to it.

### Create resource group and managed identity

```bash
az group create --name circleci-oidc-rg --location eastus
az identity create --name script-id --resource-group circleci-oidc-rg
```

### Assign Application Administrator Role

We can find an id of that role in the [documentation](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#all-roles).

```bash
principalId=$(az identity show --name script-id --resource-group circleci-oidc-rg --query principalId --output tsv)

az rest --method POST \
--uri 'https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignments' \
--body '{"@odata.type": "#microsoft.graph.unifiedRoleAssignment", "principalId": <principalId>, "roleDefinitionId": "9b895d92-2cd3-44c7-9d02-a6ac2d5ea5c3", "directoryScopeId": "/"}'
```

### Create parameters.json file

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "value": "circleci-oidc"
    },
    "federatedName": {
      "value": "circleci-federated-identity"
    },
    "organizationId": {
      "value": <circleci ORGANIZATION_ID>
    },
    "projectId": {
      "value": <circleci PROJECT_ID>
    },
    "userId": {
      "value": <circleci USER_ID>
    }
  }
}
```

### Create script.bicep
<script src="https://gist.github.com/adamkielar/00d29db4fba95ece2f0cbd87e527423b.js"></script>

```bash
az deployment group create --name oidc-001 --resource-group circleci-oidc-rg -f script.bicep -p parameters.json
```

### Create readerrole.bicep
<script src="https://gist.github.com/adamkielar/31cb47ff479681482c523934c0d7c73d.js"></script>

```bash
az deployment sub create --name oidc-002 --location eastus -f readerrole.bicep
```

<blockquote class="prompt-info">
We can extend that approach. We can create a module and create federated credentials in a for loop, passing parameters as a list of objects.
</blockquote>

## Run CircleCI job to test federated identity.

Create config.yml in .circleci folder in your git repository.
```yaml
version: 2.1

orbs:
  azure-cli: circleci/azure-cli@1.2.2

jobs:
  circleci-oidc:
    parameters:
      azure-sp:
        type: env_var_name
        default: AZURE_SP
      azure-sp-tenant:
        type: env_var_name
        default: AZURE_SP_TENANT
    executor: azure-cli/azure-docker
    steps:
      - run:
         name: Login to Azure.
         command: |
            az login --service-principal \
            --username ${<< parameters.azure-sp >>} \
            --tenant ${<< parameters.azure-sp-tenant >>} \
            --federated-token ${CIRCLE_OIDC_TOKEN}
      - run:
         name: Check Azure account.
         command: az account show

workflows:
  main:
   jobs:
    - circleci-oidc:
       context:
        - just_oidc
```

### Add environment variables in CircleCI

* Add `AZURE_SP` (application id, echo $appId)
* Add `AZURE_SP_TENANT` (az account show --query tenantId -o tsv)

![CircleCI Secrets](/assets/post2/circleci-secrets.png)

### Add context to CircleCI

In CircleCI jobs that use at least one context, the OpenID Connect ID token is available in the environment variable `$CIRCLE_OIDC_TOKEN`.

![Context](/assets/post2/context.png)

### Push changes to git and check CircleCI

It takes time for the federated identity credential to be propagated throughout a region after being initially configured. A token request made several minutes after configuring the federated identity credential may fail because the cache is populated in the directory with old data. Due to that our job can fail with error AADSTS70021: No matching federated identity record found for presented assertion. We should double-check what we added in the Subject identifier field.
If everything is set correctly our pipeline should finish the job with success.

![Success](/assets/post2/circleci-check.png)

## Video presentation

<iframe width="560" height="315" src="https://www.youtube.com/embed/JAVlv661Plk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>