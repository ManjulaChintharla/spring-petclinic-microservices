---
page_type: sample
languages:
- java
products:
- Azure Spring Apps
description: "Deploy Spring Boot apps using Azure Spring Apps and MySQL"
urlFragment: "spring-petclinic-microservices"
---

# Deploy Spring Boot apps using Azure Spring Apps and MySQL

Azure Spring Apps enables you to easily run a Spring Boot applications on Azure.

This quickstart shows you how to deploy an existing Java Spring Cloud application to Azure. When you're finished, you can continue to manage the application via the Azure CLI or switch to using the Azure Portal.

* [Deploy Spring Boot apps using Azure Spring Apps and MySQL](#deploy-spring-boot-apps-using-azure-spring-cloud-and-mysql)
  * [What will you experience](#what-will-you-experience)
  * [What you will need](#what-you-will-need)
  * [Install the Azure CLI extension](#install-the-azure-cli-extension)
  * [Clone and build the repo](#clone-and-build-the-repo)
  * [Unit 1 - Deploy and monitor Spring Boot apps](#unit-1---deploy-and-monitor-spring-boot-apps)
  * [Unit 2 - AUTOMATE deployments using GitHub Actions](#unit-2---automate-deployments-using-github-actions)
  * [Unit 3 - Manage application secrets using Azure KeyVault](#unit-3---manage-application-secrets-using-azure-keyvault)
  * [Next Steps](#next-steps)

## What will you experience
You will:
- Build existing Spring Boot applications
- Provision an Azure Spring Apps service instance. If you prefer Terraform, you may also provision using Terraform, see [`README-terraform`](./terraform/README-terraform.md)
- Deploy applications to Azure
- Connect applications to Azure Database for MySQL using Azure AD authentication
- Open the application
- Monitor applications
- Automate deployments using GitHub Actions
- Manage application secrets using Azure KeyVault

## What you will need

In order to deploy a Java app to cloud, you need an Azure subscription. If you do not already have an Azure subscription, you can activate your [MSDN subscriber benefits](https://azure.microsoft.com/pricing/member-offers/msdn-benefits-details/) or sign up for a [free Azure account]((https://azure.microsoft.com/free/)).

In addition, you will need the following:

| [Azure CLI version 2.44.0 or higher](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest) 
| [Java 8](https://adoptium.net/temurin/releases/?version=8) 
| [Maven](https://maven.apache.org/download.cgi) 
| [Git](https://git-scm.com/)
|

> Note - The Bash shell. While Azure CLI should behave identically on all environments, shell 
semantics vary. Therefore, only bash can be used with the commands in this repo. 
To complete these repo steps on Windows, use Git Bash that accompanies the Windows distribution of 

### OR Use Azure Cloud Shell

Or, you can use the Azure Cloud Shell. Azure hosts Azure Cloud Shell, an interactive shell 
environment that you can use through your browser. You can use the Bash with Cloud Shell 
to work with Azure services. You can use the Cloud Shell pre-installed commands to run the 
code in this README without having to install anything on your local environment. To start Azure 
Cloud Shell: go to [https://shell.azure.com](https://shell.azure.com), or select the 
Launch Cloud Shell button to open Cloud Shell in your browser.

To run the code in this article in Azure Cloud Shell:

1. Start Cloud Shell.

1. Select the Copy button on a code block to copy the code.

1. Paste the code into the Cloud Shell session by selecting Ctrl+Shift+V on Windows and Linux or by selecting Cmd+Shift+V on macOS.

1. Select Enter to run the code.

## Install the Azure CLI extension

Install the Azure Spring  extension for the Azure CLI using the following command

```bash
    az extension add --name spring
```
Note - `spring` CLI extension `1.5.0` or later is a pre-requisite to enable the latest Java in-process agent for Application Insights. If you already have the CLI extension, you may need to upgrade to the latest using --

```bash
    az extension update --name spring
```

## Clone and build the repo

### Create a new folder and clone the sample app repository to your Azure Cloud account  

```bash
    mkdir source-code
    git clone https://github.com/azure-samples/spring-petclinic-microservices
```

### Change directory and build the project

```bash
    cd spring-petclinic-microservices
    mvn clean package -DskipTests
```
This will take a few minutes.

## Unit-1 - Deploy and monitor Spring Boot apps

### Prepare your environment for deployments

Create a bash script with environment variables by making a copy of the supplied template:

```bash
    cp .scripts/setup-env-variables-azure-template.sh .scripts/setup-env-variables-azure.sh
```

Open `.scripts/setup-env-variables-azure.sh` and enter the following information:

```bash
    export SUBSCRIPTION=subscription-id # customize this
    export RESOURCE_GROUP=resource-group-name # customize this
    ...
    export SPRING_CLOUD_SERVICE=azure-spring-cloud-name # customize this
    ...
    export MYSQL_SERVER_NAME=mysql-servername # customize this
    ...
    export MYSQL_SERVER_ADMIN_NAME=admin-name # customize this
    ...
    export MYSQL_SERVER_ADMIN_PASSWORD=SuperS3cr3t # customize this
    ...
```

Then, set the environment:

```bash
    source .scripts/setup-env-variables-azure.sh
```

### Login to Azure

Login to the Azure CLI and choose your active subscription. Be sure to choose the active subscription that is whitelisted for Azure Spring Apps

```bash
    az login
    az account list -o table
    az account set --subscription ${SUBSCRIPTION}
```

### Create Azure Spring App service instance

Prepare a name for your Azure Spring App service.  The name must be between 4 and 32 characters long and can contain only lowercase letters, numbers, and hyphens.  The first character of the service name must be a letter and the last character must be either a letter or a number.

Create a resource group to contain your Azure Spring App service.

```bash
    az group create --name ${RESOURCE_GROUP} \
        --location ${REGION}
```

Create an instance of Azure Spring App.

```bash
    az spring create --name ${SPRING_CLOUD_SERVICE} \
            --sku standard \
            --sampling-rate 100 \
            --resource-group ${RESOURCE_GROUP} \
            --location ${REGION}
```

The service instance will take around five minutes to deploy.

Set your default resource group name and cluster name using the following commands:

```bash
    az configure --defaults \
        group=${RESOURCE_GROUP} \
        location=${REGION} \
        spring=${SPRING_CLOUD_SERVICE}
```

### Create and configure Log Analytics Workspace

Create a Log Analytics Workspace using Azure CLI:

```bash
    az monitor log-analytics workspace create \
        --workspace-name ${LOG_ANALYTICS} \
        --resource-group ${RESOURCE_GROUP} \
        --location ${REGION}

    export LOG_ANALYTICS_RESOURCE_ID=$(az monitor log-analytics workspace show \
        --resource-group ${RESOURCE_GROUP} \
        --workspace-name ${LOG_ANALYTICS} \
        --query 'id' \
        --output tsv)

    export SPRING_CLOUD_RESOURCE_ID=$(az spring show \
        --name ${SPRING_CLOUD_SERVICE} \
        --resource-group ${RESOURCE_GROUP} \
        --query 'id' \
        --output tsv)
```

Setup diagnostics and publish logs and metrics from Spring Boot apps to Azure Log Analytics:

```bash
    az monitor diagnostic-settings create --name "send-logs-and-metrics-to-log-analytics" \
        --resource ${SPRING_CLOUD_RESOURCE_ID} \
        --workspace ${LOG_ANALYTICS_RESOURCE_ID} \
        --logs '[
             {
               "category": "ApplicationConsole",
               "enabled": true,
               "retentionPolicy": {
                 "enabled": false,
                 "days": 0
               }
             },
             {
                "category": "SystemLogs",
                "enabled": true,
                "retentionPolicy": {
                  "enabled": false,
                  "days": 0
                }
              },
             {
                "category": "IngressLogs",
                "enabled": true,
                "retentionPolicy": {
                  "enabled": false,
                  "days": 0
                 }
               }
           ]' \
           --metrics '[
             {
               "category": "AllMetrics",
               "enabled": true,
               "retentionPolicy": {
                 "enabled": false,
                 "days": 0
               }
             }
           ]'
```

### Load Spring Apps Config Server

Use the `application.yml` in the root of this project to load configuration into the Config Server in Azure Spring Apps.

```bash
    az spring config-server set \
        --config-file application.yml \
        --name ${SPRING_CLOUD_SERVICE}
```

### Create applications in Azure Spring Apps

Create 5 apps.

```bash
    az spring app create --name ${API_GATEWAY} --instance-count 1 --assign-endpoint true \
        --memory 2Gi \
        --runtime-version Java_17 \
        --jvm-options='-Xms2048m -Xmx2048m'
    
    az spring app create --name ${ADMIN_SERVER} --instance-count 1 --assign-endpoint true \
        --memory 2Gi \
        --runtime-version Java_17 \
        --jvm-options='-Xms2048m -Xmx2048m'
    
    az spring app create --name ${CUSTOMERS_SERVICE} --instance-count 1 \
        --memory 2Gi \
        --runtime-version Java_17 \
        --jvm-options='-Xms2048m -Xmx2048m'
    
    az spring app create --name ${VETS_SERVICE} --instance-count 1 \
        --memory 2Gi \
        --runtime-version Java_17 \
        --jvm-options='-Xms2048m -Xmx2048m'
    
    az spring app create --name ${VISITS_SERVICE} --instance-count 1 \
        --memory 2Gi \
        --runtime-version Java_17 \
        --jvm-options='-Xms2048m -Xmx2048m'
```

### Create MySQL Database

Create a MySQL database in Azure Database for MySQL.

```bash
    # create mysql server and provide access from Azure resources
    az mysql flexible-server create \
        --name ${MYSQL_SERVER_NAME} \
        --resource-group ${RESOURCE_GROUP} \
        --location ${REGION} \
        --admin-user ${MYSQL_SERVER_ADMIN_NAME}  \
        --admin-password ${MYSQL_SERVER_ADMIN_PASSWORD} \
        --public-access 0.0.0.0 \
        --tier Burstable \
        --sku-name Standard_B1ms \
        --storage-size 32
    
    # allow access from your dev machine for testing
    MY_IP=$(curl http://whatismyip.akamai.com)
    az mysql flexible-server firewall-rule create \
            --resource-group ${RESOURCE_GROUP} \
            --name ${MYSQL_SERVER_NAME} \
            --rule-name devMachine \
            --start-ip-address ${MY_IP} \
            --end-ip-address ${MY_IP}
    
    # create database
    az mysql flexible-server db create \
            --resource-group ${RESOURCE_GROUP} \
            --server-name ${MYSQL_SERVER_NAME} \
            --database-name ${MYSQL_DATABASE_NAME}
    
    # increase connection timeout
    az mysql flexible-server parameter set \
        --resource-group ${RESOURCE_GROUP} \
        --server ${MYSQL_SERVER_NAME} \
        --name wait_timeout \
        --value 2147483
    
    # set timezone   
    az mysql flexible-server parameter set \
        --resource-group ${RESOURCE_GROUP} \
        --server ${MYSQL_SERVER_NAME} \
        --name time_zone \
        --value "US/Pacific"
    
    # create managed identity for mysql. By assigning the identity to the mysql server, it will enable Azure AD authentication
    az identity create \
        --name ${MYSQL_IDENTITY} \
        --resource-group ${RESOURCE_GROUP} \
        --location ${REGION}

    IDENTITY_ID=$(az identity show --name ${MYSQL_IDENTITY} --resource-group ${RESOURCE_GROUP} --query id -o tsv)

    # Customer service connection
    az spring connection create mysql-flexible \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --app ${CUSTOMERS_SERVICE} \
        --deployment default \
        --tg ${RESOURCE_GROUP} \
        --server ${MYSQL_SERVER_NAME} \
        --database ${MYSQL_DATABASE_NAME} \
        --system-identity mysql-identity-id=$IDENTITY_ID \
        --client-type springboot

    # Vets service connection
    az spring connection create mysql-flexible \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --app ${VETS_SERVICE} \
        --deployment default \
        --tg ${RESOURCE_GROUP} \
        --server ${MYSQL_SERVER_NAME} \
        --database ${MYSQL_DATABASE_NAME} \
        --system-identity mysql-identity-id=$IDENTITY_ID \
        --client-type springboot 
    
    # Visits service connection
    az spring connection create mysql-flexible \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --app ${VISITS_SERVICE} \
        --deployment default \
        --tg ${RESOURCE_GROUP} \
        --server ${MYSQL_SERVER_NAME} \
        --database ${MYSQL_DATABASE_NAME} \
        --system-identity mysql-identity-id=$IDENTITY_ID \
        --client-type springboot 
```

### Deploy Spring Boot applications and set environment variables

Deploy Spring Boot applications to Azure.

```bash
az spring app deploy \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --name ${API_GATEWAY} \
        --artifact-path ${API_GATEWAY_JAR} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless 

az spring app deploy \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --name ${ADMIN_SERVER} \
        --artifact-path ${ADMIN_SERVER_JAR} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless 

az spring app deploy \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --name ${CUSTOMERS_SERVICE} \
        --artifact-path ${CUSTOMERS_SERVICE_JAR} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless 

az spring app deploy \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --name ${VETS_SERVICE} \
        --artifact-path ${VETS_SERVICE_JAR} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless 

az spring app deploy \
        --resource-group ${RESOURCE_GROUP} \
        --service ${SPRING_CLOUD_SERVICE} \
        --name ${VISITS_SERVICE} \
        --artifact-path ${VISITS_SERVICE_JAR} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless 
```

```bash
    az spring app show --name ${API_GATEWAY} --query properties.url --output tsv
```

Navigate to the URL provided by the previous command to open the Pet Clinic application.
    
![](./media/petclinic.jpg)

### Monitor Spring Boot applications

#### Use the Petclinic application and make a few REST API calls

Open the Petclinic application and try out a few tasks - view pet owners and their pets, 
vets, and schedule pet visits:

```bash
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/
```

You can also `curl` the REST API exposed by the Petclinic application. The admin REST
API allows you to create/update/remove items in Pet Owners, Pets, Vets and Visits.
You can run the following curl commands:

```bash
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/owners
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/owners/4
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/owners/ 
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/petTypes
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/owners/3/pets/4
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/owners/6/pets/8/
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/vet/vets
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/visit/owners/6/pets/8/visits
curl -X GET https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/visit/owners/6/pets/8/visits
```

#### Get the log stream for API Gateway and Customers Service

Use the following command to get the latest 100 lines of app console logs from Customers Service.

```bash
az spring app logs -n ${CUSTOMERS_SERVICE} --lines 100
```
By adding a `-f` parameter you can get real-time log streaming from the app. Try log streaming for the API Gateway app.
```bash
az spring app logs -n ${API_GATEWAY} -f
```
You can use `az spring app logs -h` to explore more parameters and log stream functionalities.

#### Open Actuator endpoints for API Gateway and Customers Service apps

Spring Boot includes a number of additional features to help you monitor and manage your application when you push it to production ([Spring Boot Actuator: Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator)). You can choose to manage and monitor your application by using HTTP endpoints or with JMX. Auditing, health, and metrics gathering can also be automatically applied to your application.

Actuator endpoints let you monitor and interact with your application. By default, Spring Boot application exposes `health` and `info` endpoints to show arbitrary application info and health information. Apps in this project are pre-configured to expose all the Actuator endpoints.

You can try them out by opening the following app actuator endpoints in a browser:

```bash
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/actuator/
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/actuator/env
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/actuator/configprops

open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/actuator
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/actuator/env
open https://${SPRING_CLOUD_SERVICE}-${API_GATEWAY}.azuremicroservices.io/api/customer/actuator/configprops
```

#### Start monitoring Spring Boot apps and dependencies - in Application Insights

Open the Application Insights created by Azure Spring Apps and start monitoring Spring Boot applications. You can find the Application Insights in the same Resource Group where
you created an Azure Spring Apps service instance.

Navigate to the `Application Map` blade:
![](./media/distributed-tracking-new-ai-agent.jpg)

Navigate to the `Performance` blade:
![](./media/petclinic-microservices-performance.jpg)

Navigate to the `Performance/Dependencies` blade - you can see the performance number for dependencies, 
particularly SQL calls:
![](./media/petclinic-microservices-insights-on-dependencies.jpg)

Click on a SQL call to see the end-to-end transaction in context:
![](./media/petclinic-microservices-end-to-end-transaction-details.jpg)

Navigate to the `Failures/Exceptions` blade - you can see a collection of exceptions:
![](./media/petclinic-microservices-failures-exceptions.jpg)

Click on an exception to see the end-to-end transaction and stacktrace in context:
![](./media/end-to-end-transaction-details.jpg)

Navigate to the `Metrics` blade - you can see metrics contributed by Spring Boot apps, 
Spring Cloud modules, and dependencies. 
The chart below shows `gateway-requests` (Spring Cloud Gateway), `hikaricp_connections`
 (JDBC Connections) and `http_client_requests`.
 
![](./media/petclinic-microservices-metrics.jpg)

Spring Boot registers a lot number of core metrics: JVM, CPU, Tomcat, Logback... 
The Spring Boot auto-configuration enables the instrumentation of requests handled by Spring MVC.
All those three REST controllers `OwnerResource`, `PetResource` and `VisitResource` have been instrumented by the `@Timed` Micrometer annotation at class level.

* `customers-service` application has the following custom metrics enabled:
  * @Timed: `petclinic.owner`
  * @Timed: `petclinic.pet`
* `visits-service` application has the following custom metrics enabled:
  * @Timed: `petclinic.visit`

You can see these custom metrics in the `Metrics` blade:
![](./media/petclinic-microservices-custom-metrics.jpg)

You can use the Availability Test feature in Application Insights and monitor
the availability of applications:
![](./media/petclinic-microservices-availability.jpg)

Navigate to the `Live Metrics` blade - you can see live metrics on screen with low latencies < 1 second:
![](./media/petclinic-microservices-live-metrics.jpg)

#### Start monitoring Petclinic logs and metrics in Azure Log Analytics

Open the Log Analytics that you created - you can find the Log Analytics in the same Resource Group where you created an Azure Spring Apps service instance.

In the Log Analyics page, selects `Logs` blade and run any of the sample queries supplied below for Azure Spring Apps.

Type and run the following Kusto query to see application logs:

```sql
    AppPlatformLogsforSpring 
    | where TimeGenerated > ago(24h) 
    | limit 500
    | sort by TimeGenerated
```

Type and run the following Kusto query to see `customers-service` application logs:
```sql
    AppPlatformLogsforSpring 
    | where AppName has "customers"
    | limit 500
    | sort by TimeGenerated
```

Type and run the following Kusto query  to see errors and exceptions thrown by each app:
```sql
    AppPlatformLogsforSpring 
    | where Log contains "error" or Log contains "exception"
    | extend FullAppName = strcat(ServiceName, "/", AppName)
    | summarize count_per_app = count() by FullAppName, ServiceName, AppName, _ResourceId
    | sort by count_per_app desc 
    | render piechart
```

Type and run the following Kusto query to see all in the inbound calls into Azure Spring Apps:
```sql
    AppPlatformIngressLogs
    | project TimeGenerated, RemoteAddr, Host, Request, Status, BodyBytesSent, RequestTime, ReqId, RequestHeaders
    | sort by TimeGenerated
```

Type and run the following Kusto query to see all the logs from the managed Spring Cloud
Config Server managed by Azure Spring Apps:
```sql
    AppPlatformSystemLogs
    | where LogType contains "ConfigServer"
    | project TimeGenerated, Level, LogType, ServiceName, Log
    | sort by TimeGenerated
```

Type and run the following Kusto query to see all the logs from the managed Spring Cloud
Service Registry managed by Azure Spring Apps:
```sql
    AppPlatformSystemLogs
    | where LogType contains "ServiceRegistry"
    | project TimeGenerated, Level, LogType, ServiceName, Log
    | sort by TimeGenerated
```

## Unit-2 - Automate deployments using GitHub Actions

### Prerequisites

To get started with deploying this sample app from GitHub Actions, please:
1. Complete the sections above with your MySQL, Azure Spring Apps instances and apps created.
2. Fork this repository and turn on GitHub Actions in your fork

### Prepare GitHub Federated Credentials

You can follow the steps described [here](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure) to create a GitHub Federated Credentials in Azure Active Directory.

Create an Azure Active Directory application:

```bash
AZURE_CLIENT_ID=$(az ad app create --display-name github-petclinic-actions --query appId --output tsv)
GITHUB_OBJECTID=$(az ad app show --id $AZURE_CLIENT_ID --query id --output tsv)
```
```

Create a service principal for the application:

```bash
ASSIGNEE_OBJECTID=$(az ad sp create --id $AZURE_CLIENT_ID --query id --output tsv)
```

Create a service principle with enough scope/role to manage your Azure Spring Apps instance. Following example assigns contributor role on the resource group hosting the infrastructure:

```bash
az role assignment create --role contributor --subscription ${SUBSCRIPTION} --assignee-object-id  $ASSIGNEE_OBJECTID --assignee-principal-type ServicePrincipal --scope /subscriptions/${SUBSCRIPTION}/resourceGroups/${RESOURCE_GROUP}
```

Add federated credentials for GitHub in Azure AD.

```bash
az rest --method POST --uri 'https://graph.microsoft.com/beta/applications/<GITHUB_OBJECTID>/federatedIdentityCredentials' --body '{"name":"<CREDENTIAL-NAME>","issuer":"https://token.actions.githubusercontent.com","subject":"repo:<OWNER>/spring-petclinic-microservices:ref:refs/heads/azure","description":"Testing","audiences":["api://AzureADTokenExchange"]}'
```

In the previous command, replace the following values:
CREDENTIAL-NAME: The name of the credential. This is the name that will appear in AAD portal.
GITHUB_OBJECTID: The object ID of the service principal created in the previous step.
OWNER: The owner of the GitHub repository hosting the code. So, if you forked the repository, the owner is your GitHub username.

```bash
az rest --method POST --uri 'https://graph.microsoft.com/beta/applications/00000000-0000-0000-0000-000000000000/federatedIdentityCredentials' --body '{"name":"github-petclinic-actions","issuer":"https://token.actions.githubusercontent.com","subject":"repo:Azure-Samples/spring-petclinic-microservices:ref:refs/heads/azure","description":"Testing","audiences":["api://AzureADTokenExchange"]}'
```

The results in the command line will look like this:

```bash
{
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#applications('000000000-0000-0000-0000-000000000000')/federatedIdentityCredentials/$entity",
  "audiences": [
    "api://AzureADTokenExchange"
  ],
  "description": "Testing",
  "id": "000000000-0000-0000-0000-000000000000",
  "issuer": "https://token.actions.githubusercontent.com",
  "name": "github-petclinic-actions",
  "subject": "repo:Azure-Samples/spring-petclinic-microservices:ref:refs/heads/azure"
}
```

It will look like this in Azure AD portal:
![Federated Credentials](./media/azuread-github-federated-credential.png)

> [!NOTE] It can take few minutes to refresh the federated credentials in Azure AD portal.

### Prepare GitHub Secrets

The actions expects the following secrets to be set in your GitHub repository:

* AZURE_TENANT_ID: The tenant ID of the Azure subscription hosting the Azure Spring Apps instance.
* AZURE_SUBSCRIPTION_ID: The subscription ID of the Azure subscription hosting the Azure Spring Apps instance.
* AZURE_CLIENT_ID: The client ID of the Azure Active Directory application created in the previous step.
* RESOURCE_GROUP: The resource group hosting the Azure Spring Apps instance.
* SPRING_APPS_SERVICE_NAME: The name of the Azure Spring Apps instance.

As you can see, there are not real confidential values, as even if someone gets access to them, they can't do anything with them. They can be replaced by environment variables.

You will see GitHub Actions triggered to build and deploy all the apps in the repo to your Azure Spring Apps instance.
![](./media/automate-deployments-using-github-actions.png)

## Unit-3 - Enable Managed Identities for applications in Azure Spring Apps

Enable System Assigned Identities for applications and export identities to environment.

```bash
    az spring app identity assign --name ${CUSTOMERS_SERVICE}
    export CUSTOMERS_SERVICE_IDENTITY=$(az spring app show --name ${CUSTOMERS_SERVICE} --query 'identity.principalId' --output tsv)
    
    az spring app identity assign --name ${VETS_SERVICE}
    export VETS_SERVICE_IDENTITY=$(az spring app show --name ${VETS_SERVICE} --query 'identity.principalId' --output tsv)
    
    az spring app identity assign --name --name ${VISITS_SERVICE}
    export VISITS_SERVICE_IDENTITY=$(az spring app show --name ${VISITS_SERVICE} --query 'identity.principalId' --output tsv)
```

### Activate applications to use passwordless access to MySql

Configuration repo contains a profile for passwordless access to MySql. To activate it, we need to add the following environment variable `SPRING_PROFILES_ACTIVE=passwordless`

```bash
    az spring app update --name ${CUSTOMERS_SERVICE} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless

    az spring app update --name ${VETS_SERVICE} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless
        
    az spring app update --name ${VISITS_SERVICE} \
        --jvm-options='-Xms2048m -Xmx2048m' \
        --env SPRING_PROFILES_ACTIVE=passwordless
```

## Next Steps

In this quickstart, you've deployed an existing Spring Boot-based app using Azure CLI, Terraform and GitHub Actions. To learn more about Azure Spring Apps, go to:

- [Azure Spring Apps](https://azure.microsoft.com/products/spring-apps/)
- [Azure Spring Apps docs](https://learn.microsoft.com/azure/developer/java/)
- [Deploy Spring microservices from scratch](https://github.com/microsoft/azure-spring-cloud-training)
- [Deploy existing Spring microservices](https://github.com/Azure-Samples/azure-spring-cloud)
- [Azure for Java Cloud Developers](https://learn.microsoft.com/azure/developer/java/)
- [Spring Cloud Azure](https://spring.io/projects/spring-cloud-azure)
- [Spring Cloud](https://spring.io/projects/spring-cloud)

## Credits

This Spring microservices sample is forked from 
[spring-petclinic/spring-petclinic-microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) - see [Petclinic README](./README-petclinic.md). 

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
