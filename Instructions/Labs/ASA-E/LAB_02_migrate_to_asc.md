---
lab:
    Title: 'Challenge 02: Migrate a Spring Apps application to Azure'
    Learn module: 'Learn Module 2: Migrate a Spring Apps application to Azure'
---

# Challenge 02: Migrate a Spring Apps application to Azure

# Student manual

## Challenge scenario

You have established a plan for migrating the Spring Petclinic application to Azure Spring Apps. It is now time to perform the actual migration of the Spring Petclinic application components.

## Objectives

After you complete this challenge, you will be able to:

- Create an Azure Spring Apps Enterprise service
- Set up the config repository
- Set up the Application Configuration Service for Azure Spring Apps Enterprise
- Create an Azure MySQL Database service
- Deploy the Spring Petclinic app components to the Spring Apps service
- Provide a publicly available endpoints for the Spring Petclinic application
- Test the application through the publicly available endpoints

The below image illustrates the end state you will be building in this challenge.

![Challenge 2 architecture](./images/asa-openlab-2.png)

## Challenge Duration

- **Estimated Time**: 120 minutes

## Instructions

During this challenge, you will:

- Create an Azure Spring Apps Enterprise service
- Set up the config repository
- Set up the Application Configuration Service for Azure Spring Apps Enterprise
- Create an Azure MySQL Database service
- Deploy the Spring Petclinic app components to the Spring Apps service
- Provide a publicly available endpoints for the Spring Petclinic application
- Test the application through the publicly available endpoints

> **Note**: Follow the steps in the [install instructions](../../install.md) to set up this lab on your platform of choice.

### Create an Azure Spring Apps Enterprise service

As the first step, you will create an Azure Spring Apps Enterprise Service instance. You will use for this purpose Azure CLI. If you are interested in accomplishing this programmatically, review the Microsoft documentation that describes the provisioning process.

- [Quickstart: Provision Azure Spring Apps using Azure CLI](https://learn.microsoft.com/azure/spring-apps/quickstart-deploy-infrastructure-vnet-azure-cli?tabs=azure-spring-apps-enterprise)

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, open the Git Bash window and, from the Git Bash prompt, run the following command to sign in to your Azure subscription:

   ```bash
   az login
   ```

    > **Note**: In case you are running this lab in a GitHub codespace, use `az login --use-device-code`.

1. Executing the command will automatically open a web browser window prompting you to authenticate. Once prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab and close the web browser window.

1. Make sure that you are logged in to the right subscription for the consecutive commands.

   ```bash
   az account list -o table
   ```

1. If in the above statement you don't see the right account being indicated as your default one, change your environment to the right subscription with the following command, replacing the `<subscription-id>`.

   ```bash
   az account set --subscription <subscription-id>
   ```

1. Run the following commands to create a resource group that will contain all of your resources (replace the `<azure-region>` placeholder with the name of any Azure region in which you can create an Enterprise SKU instance of the Azure Spring Apps service and an Azure Database for MySQL Flexible Server instance, see [this page](https://azure.microsoft.com/global-infrastructure/services/?products=mysql,spring-apps&regions=all) for regional availability details of those services):

   ```bash
   UNIQUEID=$(openssl rand -hex 3)
   APPNAME=petclinic
   RESOURCE_GROUP=rg-$APPNAME-$UNIQUEID
   LOCATION=<azure-region>
   az group create -g $RESOURCE_GROUP -l $LOCATION
   ```
1. Run the following command to add and upgrade the spring extension.

   ```bash
   az extension add --upgrade --name spring
   ``` 

1. Make sure you don't have the previous `spring-cloud` extension installed.    

   ```bash
   az extension list
   az extension remove --name spring-cloud
   ```

1. Also register the `Microsoft.SaaS` provider. It might take some time for this provider to register, so check regularly with the second statement whether it indicates the provider is successfully installed before proceeding with the next statements.

   ```bash
   az provider register --namespace Microsoft.SaaS
   az provider show -n Microsoft.SaaS --query registrationState
   ```

1. Accept the license terms of the Spring Apps Enterprise tier.

   ```bash
   az term accept \
       --publisher vmware-inc \
       --product azure-spring-cloud-vmware-tanzu-2 \
       --plan asa-ent-hr-mtr
   ```

1. Run the following commands to create an instance of the enterprise SKU of the Azure Spring Apps service. Note that the name of the service needs to be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens.

   ```bash
   SPRING_APPS_SERVICE=sa-$APPNAME-$UNIQUEID
   az spring create \
       --resource-group $RESOURCE_GROUP \
       --name $SPRING_APPS_SERVICE \
       --sku enterprise \
       --enable-application-configuration-service \
       --enable-service-registry \
       --enable-gateway \
       --enable-api-portal
   ```

   > **Note**: This will also create for you an Application Insights resource. 

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Run the following command to set your default resource group name and Spring Apps service name. By setting these defaults, you don't need to repeat these names in the subsequent commands.

   ```bash
   az config set defaults.group=$RESOURCE_GROUP defaults.spring=$SPRING_APPS_SERVICE
   ```

1. Open a web browser window and navigate to the Azure portal. If prompted, sign in using the user account that has the Owner role in the target Azure subscription that you will use in this lab.

1. In the Azure portal, use the **Search resources, services, and docs** text box to search for and navigate to the resource group you just created.

1. On the resource group overview pane, verify that the resource group contains an Azure Spring Apps instance.

   > **Note**: In case you don't see the Azure Spring Apps service in the overview list of the resource group, select the **Refresh** toolbar button to refresh the view of the resource groups.

   > **Note**: You will notice an Application Insights resource also was created in your resource group. You will use this in one of the next labs.

1. Select the Azure Spring Apps instance and, in the vertical navigation menu, in the **Settings** section, select **Apps**. Note that the instance does not include any spring apps at this point. You will perform the app deployment later in this exercise.

</details>

### Set up the config repository


Azure Spring Apps Enterprise service provides an Application Configuration Service for the use of Spring apps. As part of its setup, you need to link it to a git repo. The current configuration used by the Spring microservices resides in the [config folder of the GitHub repository of this lab](https://github.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/tree/master/config). You will need to create your own private git repo in this exercise, since, in one of its steps, you will be changing some of the configuration settings.

As part of the setup process, you need to create a Personal Access Token (PAT) in your GitHub repo and make it available to the Application Configuration Service. It is important that you make note of the PAT after it has been created.

- [Guidance for creating a PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

<details>
<summary>hint</summary>
<br/>

1. On your lab computer, in your web browser, navigate to your GitHub account, navigate to the **Repositories** page and create a new private repository named **spring-petclinic-microservices-config**.

   > **Note**: Make sure to configure the repository as private. In one of the exercises this config repository will contain a secret value.

1. To create a PAT, select the avatar icon in the upper right corner, and then select **Settings**.

1. At the bottom of the vertical navigation menu, select **Developer settings**, select **Personal access tokens**, and then select **Generate new token**.

1. On the **New personal access token** page, in the **Note** text box, enter a descriptive name, such as **spring-petclinic-config-server-token**.

   > **Note**: There is a new **Beta** experience available on GitHub for more fine-grained access tokens. This experience will create a token with a more limited scope than full repository scope (which basically gives access to all your repositories). The lab will work as well with a more fine-grained token, in that case, in the **Fine-grained tokens (Beta)** token creation page, choose for **Only select repositories** and select your config repository. For the **Repository permissions** select for the **Contents** the **Read-only** access level. You can use this fine-grained token when you configure your config-server on Azure Spring Apps. We recommend you create a second token in case you also need a personal access token for interacting with the repositories from the Git Bash prompt.

1. In the **Select scopes** section, select **repo** and then select **Generate token**.

1. Record the generated token. You will need it in this and subsequent labs.

   > **Note**: You can check the validity of your token with the following statement: `curl -XGET -H 'authorization: token <token_value>' 'https://api.github.com/repos/<user_name>/spring-petclinic-microservices-config'`. This statement should succeed. If it does not, redo the above steps for generating the PAT token.

1. From the Git Bash prompt, change the current directory to the **workspaces** folder. Next, clone the newly created GitHub repository by typing `git clone `, pasting the clone URL you copied into Clipboard in the previous step, and entering the PAT string followed by the `@` symbol in front of `github.com`. In case you haven't already you can also clone here your code repository.

   ```bash
   cd ~/workspaces
   # Clone config repo
   git clone https://<token>@github.com/<your-github-username>/spring-petclinic-microservices-config.git
    
   # Clone source code repo
   git clone https://<token>@github.com/<your-github-username>/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps.git

   ```

    > **Note**: Make sure to replace the `<token>` and `<your-github-username>` placeholders in the URL listed above with the value of the GitHub PAT and your GitHub user name when running the `git clone` command.

1. From the Git Bash prompt, change the current directory to the newly created **spring-petclinic-microservices-config** folder and run the following commands to copy all the config server configuration yaml files from the [config folder of this labs' GitHub repository](https://github.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/tree/master/config) to the local folder on your lab computer.

   ```bash
   cd spring-petclinic-microservices-config
   curl -o admin-server.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/admin-server.yml
   curl -o api-gateway.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/api-gateway.yml
   curl -o application.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/application.yml
   curl -o customers-service.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/customers-service.yml
   curl -o discovery-server.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/discovery-server.yml
   curl -o tracing-server.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/tracing-server.yml
   curl -o vets-service.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/vets-service.yml
   curl -o visits-service.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/visits-service.yml
   curl -o messaging-emulator.yml https://raw.githubusercontent.com/MicrosoftLearning/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/master/config/messaging-emulator.yml
   ```

1. From the Git Bash prompt, run the following commands to commit and push your changes to your private GitHub repository.

   ```bash
   git add .
   git commit -m 'added base config'
   git push
   ```

1. In your web browser, refresh the page of the newly created **spring-petclinic-microservices-config** repository and double check that all the configuration files are there.

</details>

### Set up the Application Configuration Service for Azure Spring Apps Enterprise
    
Once you completed the initial update of your git repository hosting the server configuration, you need to set up the Application Configuration Service for your Azure Spring Apps Enterprise instance. 

- [Externalize configuration with Application Configuration Service](https://learn.microsoft.com/azure/spring-apps/quickstart-deploy-apps-enterprise#externalize-configuration-with-application-configuration-service).
- [Use Application Configuration Service for Tanzu](https://learn.microsoft.com/azure/spring-apps/how-to-enterprise-application-configuration-service?tabs=Azure-CLI).

<details>
<summary>hint</summary>
<br/>

1. Switch to the Git Bash prompt and run the following commands to set the environment variables hosting your GitHub repository and GitHub credentials (replace the `<git-repository>`, `<git-username>`, and `<git-PAT>` placeholders with the URL of your GitHub repository, the name of your GitHub user account, and the newly generated PAT value, respectively).

   > **Note**: The URL of the GitHub repository should be in the format `https://github.com/<your-github-username>/spring-petclinic-microservices-config.git`, where the `<your-github-username>` placeholder represents your GitHub user name.

   ```bash
   GIT_REPO=<git-repository>
   GIT_USERNAME=<git-username>
   GIT_PASSWORD=<git-PAT>
   ```

1. To set up the Application Configuration Service such that it points to your GitHub repository, from the Git Bash prompt, run the following command.

   ```bash
   az spring application-configuration-service git repo add \
       --resource-group $RESOURCE_GROUP \
       --name spring-petclinic-config \
       --service $SPRING_APPS_SERVICE \
       --label main \
       --patterns "api-gateway,customers-service,vets-service,visits-service,admin-server" \
       --uri $GIT_REPO \
       --password $GIT_PASSWORD \
       --username $GIT_USERNAME
   ```

   > **Note**: In case you are using a branch other than `main` in your config repo, you can change the branch name with the `label` parameter.

   > **Note**: Wait for the operation to complete. This might take about 2 minutes.

</details>

### Create an Azure MySQL Database service

You now have the compute service that will host your applications and the config server that will be used by your migrated application. Before you start deploying individual microservices as Azure Spring Apps applications, you need to first create an Azure Database for MySQL Single Server-hosted database for them. To accomplish this, you can use the following guidance:

- [Create MySQL Flexible Server and Database](https://learn.microsoft.com/azure/mysql/flexible-server/quickstart-create-server-cli).

You will also need to update the config for your applications to use the newly provisioned MySQL Server to authorize access to your private GitHub repository. This will involve updating the application.yml config file in your private git config repo with the values provided in the MySQL Server connection string.

<details>
<summary>hint</summary>
<br/>

1. Run the following commands to create an instance of MySQL Flexible server. Note that the name of the server must be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens. In addition, replace the `<myadmin-password>` placeholder with a complex password and record its value.

   ```bash
   MYSQL_SERVER_NAME=mysql-$APPNAME-$UNIQUEID
   MYSQL_ADMIN_USERNAME=myadmin
   MYSQL_ADMIN_PASSWORD=<myadmin-password>
   DATABASE_NAME=petclinic
   
    az mysql flexible-server create \
        --admin-user myadmin \
        --admin-password ${MYSQL_ADMIN_PASSWORD} \
        --name ${MYSQL_SERVER_NAME} \
        --resource-group ${RESOURCE_GROUP} 
   ```

   > **Note**: During the creation you will be asked whether access for your IP address should be added and whether access for all IP's should be added. Answer `n` for no on both questions.

   > **Note**: Wait for the provisioning to complete. This might take about 3 minutes.

1. Once the Azure Database for MySQL Single Server instance gets created, it will output details about its settings. In the output, you will find the server connection string. Record its value since you will need it later in this exercise.

1. Run the following commands to create a database in the Azure Database for MySQL Single Server instance.

   ```bash
    az mysql flexible-server db create \
        --server-name $MYSQL_SERVER_NAME \
        --resource-group $RESOURCE_GROUP \
        -d $DATABASE_NAME
   ```

1. You will also need to allow connections to the server from Azure Spring Apps. For now, to accomplish this, you will create a server firewall rule to allow inbound traffic from all Azure Services. This way your apps running in Azure Spring Apps will be able to reach the MySQL database providing them with persistent storage. In one of the upcoming exercises, you will restrict this connectivity to limit it exclusively to the apps hosted by your Azure Spring Apps instance.

   ```bash
    az mysql flexible-server firewall-rule create \
        --rule-name allAzureIPs \
        --name ${MYSQL_SERVER_NAME} \
        --resource-group ${RESOURCE_GROUP} \
        --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
   ```

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the _application.yml_ file. Replace the full contents of the _application.yml_ file with the contents of [this application.yml](../../../config/02_application.yml) file. The updated _application.yml_ file includes the following changes:

   * It changes the default `spring.sql.init` values to use `mysql` configuration on lines 15 to 19.
   * It adds a `spring.datasource` property for your mysql database on lines 10 to 14.
   * It adds extra `eureka` config on lines 61 to 66.
   * It removes the `chaos-monkey` and `mysql` profiles.

1. In the part you pasted, update the values of the target datasource endpoint on line 12, the corresponding admin user account on line 13, and its password on line 14 to match your configuration. Set these values by using the information in the Azure Database for MySQL Flexible Server connection string you recorded earlier in this task.

1. Save the changes and push the updates you made to the **application.yml** file to your private GitHub repo by running the following commands from the Git Bash prompt:

   ```bash
   git add .
   git commit -m 'azure mysql info'
   git push
   ```

</details>

   > **Note**: At this point, the admin account user name and password are stored in clear text in the application.yml config file. In one of upcoming exercises, you will remediate this potential vulnerability by removing clear text credentials from your configuration.

### Deploy the Spring Petclinic app components to the Spring Apps service Enterprise

You now have the compute and data services available for deployment of the components of your applications, including `spring-petclinic-admin-server`, `spring-petclinic-customers-service`, `spring-petclinic-vets-service`, `spring-petclinic-visits-service` and `spring-petclinic-api-gateway`. In this task, you will deploy these components as microservices to the Azure Spring Apps service. You will not be deploying the `spring-petclinic-config-server` and `spring-petclinic-discovery-server` to Azure Spring Apps, since these will be provided to you by the platform. To perform the deployment, you can use the following guidance:

- [Quickstart: Build and deploy apps to Azure Spring Apps using the Enterprise plan](https://learn.microsoft.com/azure/spring-apps/quickstart-deploy-apps-enterprise).

   > **Note**: The `spring-petclinic-api-gateway` and `spring-petclinic-admin-server` will have a public endpoint assigned to them.

<details>
<summary>hint</summary>
<br/>

1. In the src directory parent **pom.xml** file double check the version number on line 9.

    ```bash
       <parent>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-parent</artifactId>
           <version>3.0.2</version>
       </parent>
    ```

1. From the Git Bash window, set a `VERSION` environment variable to this version number `3.0.2`.

   ```bash
   VERSION=3.0.2
   ```

1. You will start by building all the microservice of the spring petclinic application. To accomplish this, run `mvn clean package` in the root directory of the application.

   ```bash
   cd ~/workspaces/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/src
   mvn clean package -DskipTests
   ```

1. Verify that the build succeeds by reviewing the output of the `mvn clean package -DskipTests` command, which should have the following format:

   ```bash
   [INFO] ------------------------------------------------------------------------
   [INFO] Reactor Summary for spring-petclinic-microservices 3.0.2:
   [INFO] 
   [INFO] spring-petclinic-microservices ..................... SUCCESS [  0.249 s]
   [INFO] spring-petclinic-admin-server ...................... SUCCESS [ 16.123 s]
   [INFO] spring-petclinic-customers-service ................. SUCCESS [  6.749 s]
   [INFO] spring-petclinic-vets-service ...................... SUCCESS [  4.845 s]
   [INFO] spring-petclinic-visits-service .................... SUCCESS [  5.063 s]
   [INFO] spring-petclinic-config-server ..................... SUCCESS [  1.777 s]
   [INFO] spring-petclinic-discovery-server .................. SUCCESS [  2.563 s]
   [INFO] spring-petclinic-api-gateway ....................... SUCCESS [ 15.582 s]
   [INFO] ------------------------------------------------------------------------
   [INFO] BUILD SUCCESS
   [INFO] ------------------------------------------------------------------------
   [INFO] Total time:  55.901 s
   [INFO] Finished at: 2023-06-02T14:07:49Z
   [INFO] ------------------------------------------------------------------------
   ```

1. You will set a couple of environment variables for the name of each app.

   ```bash
   API_GATEWAY=api-gateway
   ADMIN_SERVER=admin-server
   CUSTOMERS_SERVICE=customers-service
   VETS_SERVICE=vets-service
   VISITS_SERVICE=visits-service
   ```

1. For each application you will now create an app on Azure Spring Apps service. You will start with the `api-gateway`. To deploy it, from the Git Bash prompt, run the following command:

   ```bash
   az spring app create \
       --name $API_GATEWAY \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --assign-endpoint true
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 5 minutes.

1. Next, bind the application to the Application Configuration Service.

   ```bash
   az spring application-configuration-service bind --app ${API_GATEWAY}
   ```

1. And also bind the app to the service registry.

   ```bash
   az spring service-registry bind --app ${API_GATEWAY}
   ```

1. Next deploy the jar file to this newly created app by running the following command from the Git Bash prompt:

   ```bash
   API_GATEWAY_JAR=spring-petclinic-api-gateway/target/spring-petclinic-api-gateway-$VERSION.jar
   az spring app deploy --name ${API_GATEWAY} \
       --config-file-patterns ${API_GATEWAY} \
       --artifact-path ${API_GATEWAY_JAR} \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   ```

1. In the same way create an app for the `admin-server` microservice, bind it and deploy it:

   ```bash
   az spring app create \
       --name $ADMIN_SERVER \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --assign-endpoint true
   az spring application-configuration-service bind --app ${ADMIN_SERVER}
   az spring service-registry bind --app ${ADMIN_SERVER}
   ADMIN_SERVER_JAR=spring-petclinic-admin-server/target/spring-petclinic-admin-server-$VERSION.jar
   az spring app deploy --name ${ADMIN_SERVER} \
       --config-file-patterns ${ADMIN_SERVER} \
       --artifact-path ${ADMIN_SERVER_JAR} \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   ```

   > **Note**: Wait for each operation to complete. This might take about 5 minutes.

1. Next, you will create, bind and deploy an app for the `customers-service` microservice, without assigning an endpoint:

   ```bash
   az spring app create \
       --name $CUSTOMERS_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   az spring application-configuration-service bind --app ${CUSTOMERS_SERVICE}
   az spring service-registry bind --app ${CUSTOMERS_SERVICE}
   CUSTOMERS_SERVICE_JAR=spring-petclinic-customers-service/target/spring-petclinic-customers-service-$VERSION.jar
   az spring app deploy --name ${CUSTOMERS_SERVICE} \
       --config-file-patterns ${CUSTOMERS_SERVICE} \
       --artifact-path ${CUSTOMERS_SERVICE_JAR} \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   ```

   > **Note**: Wait for each operation to complete. This might take about 5 minutes.

1. Once deployed, take a look at the `customers-service` logs to make sure the configuration gets picked up correctly and there are no errors on startup.

   ```bash
   az spring app logs --name ${CUSTOMERS_SERVICE} --follow 
   ```

   > **Note**: In case you see no errors, you can escape out of the log statement with `Ctrl+C` and you can proceed with the next steps. In case you see errors, review the steps you executed and retry. The [LabTips file](../../../LabTips.md) also contains steps on how to recover from errors.

1. Next, you will create, bind and deploy an app for the `visits-service` microservice, also without an endpoint assigned:

   ```bash
   az spring app create \
       --name $VISITS_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE 
   az spring application-configuration-service bind --app ${VISITS_SERVICE}
   az spring service-registry bind --app ${VISITS_SERVICE}
   VISITS_SERVICE_JAR=spring-petclinic-visits-service/target/spring-petclinic-visits-service-$VERSION.jar
   az spring app deploy --name ${VISITS_SERVICE} \
       --config-file-patterns ${VISITS_SERVICE} \
       --artifact-path ${VISITS_SERVICE_JAR} \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   ```

   > **Note**: Wait for each operation to complete. This might take about 5 minutes.

1. To conclude, you will create, bind and deploy an app for the `vets-service` microservice, again without an endpoint assigned:

   ```bash
   az spring app create \
       --name $VETS_SERVICE \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE 
   az spring application-configuration-service bind --app ${VETS_SERVICE}
   az spring service-registry bind --app ${VETS_SERVICE}
   VETS_SERVICE_JAR=spring-petclinic-vets-service/target/spring-petclinic-vets-service-$VERSION.jar
   az spring app deploy --name ${VETS_SERVICE} \
       --config-file-patterns ${VETS_SERVICE}  \
       --artifact-path ${VETS_SERVICE_JAR} \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE
   ```

   > **Note**: Wait for each operation to complete. This might take about 5 minutes.

</details>

### Test the application through the publicly available endpoint

If any of the deployments failed, you can check the logs of a specific app using the following CLI statement (using `customers-service` as an example):

```bash
az spring app logs --name ${CUSTOMERS_SERVICE} --follow
```

Now that you have deployed all of your microservices, verify that the application is accessible via a web browser.

<details>
<summary>hint</summary>
<br/>

1. To list all deployed apps, from the Git Bash shell, run the following CLI statement, which will also list all publicly accessible endpoints:

   ```bash
   az spring app list --service $SPRING_APPS_SERVICE \
                      --resource-group $RESOURCE_GROUP \
                      --output table
   ```

1. You can also `grep` the URL that got assigned to the `api-gateway` service. 

   ```bash
   az spring app show --name ${API_GATEWAY} | grep url
   ```

1. Alternatively, you can switch to the web browser window displaying the Azure portal interface, navigate to your Azure Spring Apps instance and select **Apps** from the vertical navigation menu. In the list of apps, select **api-gateway**, on the **api-gateway | Overview** page, note the value of the **URL** property.

1. Open another web browser tab and navigate to the URL of the api-gateway endpoint to display the application web interface.

1. You can also navigate to the URL of the admin-server to see insight information of your microservices.

</details>

#### Review

In this exercise, you migrated your existing Spring Petclinic microservices application to Azure Spring Apps Enterprise.
