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
We will use [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/), [Azure Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/overview) and [Azure Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/) to provision resources.

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
System-assigned service principal is not supported as a SQL database user.
For purpose of this scenario we will add it as an admin but please remember that it is not a best practice.

We need to setup random App Service and SQL Server name to avoid errors while provisioning resources.
```bash
appservice="misqlapp$RANDOM"
sqlserver="misqlserver$RANDOM"
``` 

We will use a custom image to test our connection.
```bash
az webapp create \
--resource-group mi-sql-rg \
--plan misqlplan \
--name $appservice \
--deployment-container-image-name adamkielar/mi-sql:v1.0.0
```

### Assign system-assigned managed identity to the App service
``` bash
az webapp identity assign --resource-group mi-sql-rg --name $appservice
```

### AAD group with access to the SQL server
```bash
az ad group create --display-name MISQLADMINS --mail-nickname MISQLADMINS
```

### Add managed identity to just created AAD group
```bash
principalId=$(az webapp identity show --resource-group mi-sql-rg --name $appservice --query principalId --output tsv)
groupId=$(az ad group show --group MISQLADMINS --query id --output tsv)
az ad group member add --group $groupId --member-id $principalId
```

### SQL server with AD admin
```bash
az sql server create \
--enable-ad-only-auth \
--external-admin-principal-type Group \
--external-admin-name MISQLADMINS \
--external-admin-sid $groupId \
--resource-group mi-sql-rg \
--name $sqlserver
```

### SQL database
```bash
az sql db create \
--name misqldb \
--server $sqlserver \
--resource-group mi-sql-rg \
--edition GeneralPurpose \
--family Gen5 \
--capacity 2
```

### Allow Azure services and resources to access the database
```bash
az sql server firewall-rule create \
--resource-group mi-sql-rg \
--server $sqlserver \
--name misqldb \
--start-ip-address 0.0.0.0 \
--end-ip-address 0.0.0.0
```

### Add the necessary app settings like database name, SQL server name and website port
```bash
az webapp config appsettings set \
--resource-group mi-sql-rg \
--name $appservice \
--settings DATABASE=misqldb DBSERVER=$sqlserver.database.windows.net IDENTITY=system WEBSITES_PORT=8000
```

### Test connection:
App Service might need a moment to start up and process request.
```bash
curl https://$appservice.azurewebsites.net/api/mssql_db
```

As a response, we receive a database version.
```bash
{"mssql version":"Microsoft SQL Azure (RTM) - 12.0.2000.8 \n\tOct 18 2022 13:24:45 \n\tCopyright (C) 2022 Microsoft Corporation\n"}
```

In the application, we are using Python SQL driver - [pyodbc](https://github.com/mkleehammer/pyodbc).
When we are using system-assign identity, the database connection string looks as follows:
```bash
DRIVER={ODBC Driver 18 for SQL Server};SERVER=$sqlserver.database.windows.net,1433;DATABASE=misqldb;Authentication=ActiveDirectoryMSI
```

### Delete the App service and recreate it with a user-assigned identity
In that scenario we can add service principal as a database user and grant him read and write permissions.

```bash
az webapp delete --name $appservice --resource-group mi-sql-rg

az appservice plan create --name misqlplan --resource-group mi-sql-rg --is-linux --sku B1

az webapp create \
--resource-group mi-sql-rg \
--plan misqlplan \
--name $appservice \
--deployment-container-image-name adamkielar/mi-sql:v1.0.0

az webapp config appsettings set \
--resource-group mi-sql-rg \
--name $appservice \
--settings DATABASE=misqldb DBSERVER=$sqlserver.database.windows.net IDENTITY=user WEBSITES_PORT=8000
```

### Create identity
```bash
az identity create --name mi-sql-identity --resource-group mi-sql-rg
```

### Assign user-assigned managed identity to the App service 
We need a fully qualified resource Id of identity
``` bash
principalId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query id --output tsv)

az webapp identity assign --resource-group mi-sql-rg --name $appservice --identities $principalId
```

### Add UID appsetting which contains clientId of created identity

```bash
clientId=$(az identity show --name mi-sql-identity --resource-group mi-sql-rg --query clientId --output tsv)

az webapp config appsettings set --resource-group mi-sql-rg --name $appservice --settings UID=$clientId
```

### Add user to MISQLADMINS group
This user will have admin rights. Check [documentation](https://learn.microsoft.com/en-us/cli/azure/ad/user?view=azure-cli-latest) for possible options. You can add your current user. `User id` will be your email address.
```bash
userId=$(az ad user show --id <user id> --query id -o tsv)
```
```bash
groupId=$(az ad group show --group MISQLADMINS --query id --output tsv)
az ad group member add --group $groupId --member-id $userId
```

### Add AAD-base database user with read and write permissions.
Open Azure Cloud Shell and get access token
```powershell
$token = (Get-AzAccessToken -ResourceUrl https://database.windows.net).Token
```

Connect to database and create user
```powershell
Invoke-SqlCmd -ServerInstance "$sqlserver" `
-Database "misqldb" `
-AccessToken "$token" `
-Query "CREATE USER [mi-sql-identity] FROM EXTERNAL PROVIDER;"
```

Add read and write permissions to our user
```powershell
Invoke-SqlCmd -ServerInstance "$sqlserver" `
-Database "misqldb" `
-AccessToken "$token" `
-Query "ALTER ROLE db_datareader ADD MEMBER [mi-sql-identity]; ALTER ROLE db_datawriter ADD MEMBER [mi-sql-identity]"
```

Now we can test our connection as before.

When we are using user-assign identity, the database connection string looks as follows:
```bash
DRIVER={ODBC Driver 18 for SQL Server};SERVER=$sqlserver.database.windows.net,1433;DATABASE=misqldb;UID=<clientId>;Authentication=ActiveDirectoryMSI
```

### Delete a resource group to clean up our environment
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

Azure Bicep as of the time of writing does not support CRUD operations on MS Graph resources. We can bypass that problem using the deployment script but we will skip that step in this post and we will create AAD group and add user (for example user that we use for logging to Azure account) to that group, using Azure CLI as before. We can also create database user using Azure Cloud Shell as before.

### Create SQL server, database and firewall rule
<script src="https://gist.github.com/adamkielar/6614dc77cd75021984ed51fd9b061cf2.js"></script>

When we run below command we have to provide two parameters for this deployment, `sqlServerName`(we can get it from output of last deployment) and `groupId`(check description in sql.bicep).
```bash
az deployment group create --name misql-003 --resource-group mi-sql-rg -f sql.bicep
```

### Confirm in Azure portal if resources are created

![Azure portal](/assets/post1/mi-sql-rg.png)

Now we can curl the App Service endpoint and confirm the database connection.

### Delete a resource group
```bash
az group delete --resource-group mi-sql-rg
```

## YouTube

<iframe width="560" height="315" src="https://www.youtube.com/embed/L5eblUHIxSM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>