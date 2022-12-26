---
layout: post
title: "How to connect to Azure SQL database using managed identity"
tags: ["azure", "mssql", "managed identity"]
---

## What are the benefits of using managed identities?
1. Applications can use managed identities to obtain Azure AD tokens without having to manage credentials.
2. As developers, we do not have to manage credentials.
3. Using managed identities is free.

## Why use managed identity to connect to Azure SQL?
1. Our application can connect to a database without using a password.
2. We do not have to share passwords in our project.
3. Only users in Azure AD group can access a database so it is easy to onboard and offboard devs.

If you want to learn more about managed identities please check the [documentation](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview).

## Overview
![Diagram](/assets/post1/post-1-diagram.png)

We need to create the following resources to show how to set up a connection between the application and the database:
* App Service
* SQL database

Then we will deploy FastAPI application to test a connection.
We will use Azure CLI and Azure Bicep to provision resources.

## Create resources using Azure CLI.
### Resource Group
```bash
az group create --name mi-sql-rg --location eastus
```

### App Service Plan
```bash
az appservice plan create --name misqlplan --resource-group mi-sql-rg --is-linux --sku B1
```

### App Service using system-assigned managed identity

We will use a custom image to test our connection.
```bash
az webapp create \
--resource-group mi-sql-rg \
--plan misqlplan \
--name mi-sql-app \
--deployment-container-image-name adamkielar/mi-sql:v1.0.0
```

### Assign system-assigned managed identity to the App service
``` bash
az webapp identity assign --resource-group mi-sql-rg --name mi-sql-app
```

### Create a group in AAD that will have access to the SQL server
```bash
az ad group create --display-name MISQLADMINS --mail-nickname MISQLADMINS
```

### Add managed identity to just created AAD group
```bash
principalId=$(az webapp identity show --resource-group mi-sql-rg --name mi-sql-app --query principalId --output tsv)
groupId=$(az ad group show --group MISQLADMINS --query id --output tsv)
az ad group member add --group $groupId --member-id $principalId
```

### Create SQL server with AD admin
```bash
az sql server create \
--enable-ad-only-auth 
--external-admin-principal-type Group \
--external-admin-name MISQLADMINS \
--external-admin-sid $groupId \
--resource-group mi-sql-rg \
--name misqlserver
```

### Create database
```bash
az sql db create \
--name misqldb \
--server misqlserver \
--resource-group mi-sql-rg \
--edition GeneralPurpose \
--family Gen5 \
--capacity 2
```

### Allow Azure services and resources to access the database
```bash
az sql server firewall-rule create \
--resource-group mi-sql-rg \
--server misqlserver \
--name misqldb \
--start-ip-address 0.0.0.0 \
--end-ip-address 0.0.0.0
```

### We have to add the necessary app settings like database name, SQL server name and website port
```bash
az webapp config appsettings set \
--resource-group mi-sql-rg \
--name mi-sql-app \
--settings DATABASE=misqldb DBSERVER=misqlserver.database.windows.net IDENTITY=system WEBSITES_PORT=8000
```

### Now finally we can test our connection:
```bash
curl https://mi-sql-app.azurewebsites.net/api/mssql_db
```

As a response, we receive a database version.
```bash
{"mssql version":"Microsoft SQL Azure (RTM) - 12.0.2000.8 \n\tOct 18 2022 13:24:45 \n\tCopyright (C) 2022 Microsoft Corporation\n"}
```

In the application, we are using Python SQL driver - [pyodbc](https://github.com/mkleehammer/pyodbc).
When we are using system-assign identity, the database connection string looks as follows:
```bash
DRIVER={ODBC Driver 18 for SQL Server};SERVER=misqlserver.database.windows.net,1433;DATABASE=misqldb;Authentication=ActiveDirectoryMSI
```

### Let's delete the App service and recreate it with a user-assigned identity
```bash
az webapp delete --name mi-sql-app --resource-group mi-sql-rg

az appservice plan create --name misqlplan --resource-group mi-sql-rg --is-linux --sku B1

az webapp create \
--resource-group mi-sql-rg \
--plan misqlplan \
--name mi-sql-app \
--deployment-container-image-name adamkielar/mi-sql:v1.0.0

az webapp config appsettings set \
--resource-group mi-sql-rg \
--name mi-sql-app \
--settings DATABASE=misqldb DBSERVER=misqlserver.database.windows.net IDENTITY=user WEBSITES_PORT=8000
```

### Create identity
```bash
az identity create --name mi-sql-identity --resource-group mi-sql-rg
```

### Add managed identity to just MISQLADMINS Azure Active Directory group
```bash
principalId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query principalId --output tsv)
groupId=$(az ad group show --group MISQLADMINS --query id --output tsv)
az ad group member add --group $groupId --member-id $principalId
```

### Assign user-assigned managed identity to the App service 
We need a fully qualified resource Id of identity
``` bash
principalId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query id --output tsv)

az webapp identity assign --resource-group mi-sql-rg --name mi-sql-app --identities $principalId
```

### As the last step, we need to add UID appsetting which contains clientId of created identity

```bash
clientId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query id --output tsv)

az webapp config appsettings set --resource-group mi-sql-rg --name mi-sql-app --settings UID=$clientId
```

Now we can test our connection as before.

When we are using user-assign identity, the database connection string looks as follows:
```bash
DRIVER={ODBC Driver 18 for SQL Server};SERVER=misqlserver.database.windows.net,1433;DATABASE=misqldb;UID=<clientId>;Authentication=ActiveDirectoryMSI
```

### Let's delete a resource group to clean up our environment
```bash
az group delete --resource-group mi-sql-rg
```


## Create resources using Azure Bicep.

### Create a resource group in the subscription scope
<script src="https://gist.github.com/adamkielar/dc9fe1bd40f27642168b66677d1738d4.js"></script>

```bash
az deployment sub create --location eastus --name misql-001 -f resource-group.bicep
```

### Create Identity, App service with plan and app settings
<script src="https://gist.github.com/adamkielar/ecb77a332e5cdcca8eb5582c567c047c.js"></script>

```bash
az deployment group create --name misql-002 --resource-group mi-sql-rg -f appservice.bicep
```

Azure Bicep as of the time of writing does not support CRUD operations on MS Graph resources. We can bypass that problem using the deployment script but we will skip that step in this post and we will create AAD group and add identity to that group, using Azure CLI as before.

```bash
az ad group create --display-name MISQLADMINS --mail-nickname MISQLADMINS

principalId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query principalId --output tsv)

groupId=$(az ad group show --group MISQLADMINS --query id --output tsv)

az ad group member add --group $groupId --member-id $principalId
```

### Create SQL server, database and firewall rule
<script src="https://gist.github.com/adamkielar/6614dc77cd75021984ed51fd9b061cf2.js"></script>

```bash
az deployment group create --name misql-003 --resource-group mi-sql-rg -f sql.bicep
```

### We can confirm in Azure portal if resources are created

![Azure portal](/assets/post1/mi-sql-rg.png)

Now we can curl the App Service endpoint and confirm the database connection.

### Finally, we can delete a resource group
```bash
az group delete --resource-group mi-sql-rg
```