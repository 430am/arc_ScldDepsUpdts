---
type: docs
title: "Azure API Management gateway ARM Template"
linkTitle: "Azure API Management gateway ARM Template"
weight: 3
description: >
---

## Deploy an Azure API Management gateway on AKS using an ARM Template

The following README will guide you on how to deploy a "Ready to Go" environment so you can start using [a self-hosted Azure API Management Gateway](https://docs.microsoft.com/en-us/azure/api-management/how-to-deploy-self-hosted-gateway-azure-arc) deployed on an [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes) cluster that has been onboarded as an [Azure Arc enabled Kubernetes cluster](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/overview) using an [Azure ARM Template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview).

By the end of this guide, you will have an AKS cluster deployed with an Azure API Management gateway, a backend API and a Microsoft Windows Server 2022 (Datacenter) Azure VM, installed & pre-configured with all the required tools needed to work with the Azure API Management gateway.

> **Note: Currently, API Management self-hosted gateway on Azure Arc is in preview. The deployment time for this scenario can take ~60-90 minutes**

## Prerequisites

* Clone the Azure Arc Jumpstart repository

    ```shell
    git clone https://github.com/microsoft/azure_arc.git
    ```

* [Install or update Azure CLI to version 2.15.0 and above](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest). Use the below command to check your current installed version.

  ```shell
  az --version
  ```

* [Generate SSH Key](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed) (or use existing ssh key).

* Create Azure service principal (SP)

    To be able to complete the scenario and its related automation, Azure service principal assigned with the “Contributor” role is required. To create it, login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/).

    ```shell
    az login
    az ad sp create-for-rbac -n "<Unique SP Name>" --role contributor
    ```

    For example:

    ```shell
    az ad sp create-for-rbac -n "http://AzureArcAppSvc" --role contributor
    ```

    Output should look like this:

    ```json
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "AzureArcAppSvc",
    "name": "http://AzureArcAppSvc",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **Note: It is optional, but highly recommended, to scope the SP to a specific [Azure subscription](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest).**

## Automation Flow

For you to get familiar with the automation and deployment flow, below is an explanation.

* User is editing the ARM template parameters file (1-time edit). These parameters values are being used throughout the deployment.

* Main [_azuredeploy_ ARM template](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/azuredeploy.json) will initiate the deployment of the linked ARM templates:

  * [_clientVm_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/clientVm.json) - Deploys the client Windows VM. This is where all user interactions with the environment are made from.
  * [_logAnalytics_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/logAnalytics.json) - Deploys Azure Log Analytics workspace to support Azure API Management logs uploads.

* User remotes into client Windows VM, which automatically kicks off the [_AppServicesLogonScript_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/artifacts/AppServicesLogonScript.ps1) PowerShell script that deploy the AKS cluster and will configure Azure Arc enabled app services Kubernetes environment on the AKS cluster.

    > **Note: Notice the AKS cluster will be deployed via the PowerShell script automation.**

## Deployment

    > **Note: The deployment time for this scenario can take ~60-90 minutes**

As mentioned, this deployment will leverage ARM templates. You will deploy a single template that will initiate the entire automation for this scenario.

* The deployment is using the ARM template parameters file. Before initiating the deployment, edit the [_azuredeploy.parameters.json_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/azuredeploy.parameters.json) file located in your local cloned repository folder. An example parameters file is located [here](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/artifacts/azuredeploy.parameters.example.json).

  * _`sshRSAPublicKey`_ - Your SSH public key
  * _`spnClientId`_ - Your Azure service principal id
  * _`spnClientSecret`_ - Your Azure service principal secret
  * _`spnTenantId`_ - Your Azure tenant id
  * _`windowsAdminUsername`_ - Client Windows VM Administrator name
  * _`windowsAdminPassword`_ - Client Windows VM Password. Password must have 3 of the following: 1 lower case character, 1 upper case character, 1 number, and 1 special character. The value must be between 12 and 123 characters long.
  * _`myIpAddress`_ - Your local public IP address. This is used to allow remote RDP and SSH connections to the client Windows VM and AKS cluster.
  * _`logAnalyticsWorkspaceName`_ - Unique name for the deployment log analytics workspace.
  * _`kubernetesVersion`_ - AKS version
  * _`dnsPrefix`_ - AKS unique DNS prefix
  * _`deployAppService`_  Boolean that sets whether or not to deploy App Service plan and a Web App. For this scenario, we leave it set to _**false**_.
  * _`deployFunction`_ - Boolean that sets whether or not to deploy App Service plan and an Azure Function application. For this scenario, we leave it set to _**false**_.
    * _`deployLogicApp`_ - Boolean that sets whether or not to deploy App Service plan and an Azure Logic App. For this scenario, we leave it set to false.
  * _`deployAPIMgmt`_ - Boolean that sets whether or not to deploy a self-hosted Azure API Management gateway. For this scenario, we leave it set to _**true**_.
  * _`templateBaseUrl`_ - GitHub URL to the deployment template - filled in by default to point to [Microsoft/Azure Arc](https://github.com/microsoft/azure_arc) repository, but you can point this to your forked repo as well.
  * _`adminEmail`_ - an email address that will be used on the Azure API Management deployment to receive all system notifications.

* To deploy the ARM template, navigate to the local cloned [deployment folder](https://github.com/microsoft/azure_arc/tree/main/azure_arc_app_services_jumpstart/aks/arm_template) and run the below command:

    ```shell
    az group create --name <Name of the Azure resource group> --location <Azure Region>
    az deployment group create \
    --resource-group <Name of the Azure resource group> \
    --name <The name of this deployment> \
    --template-uri https://raw.githubusercontent.com/microsoft/azure_arc/main/azure_arc_app_services_jumpstart/aks/arm_template/azuredeploy.json \
    --parameters <The *azuredeploy.parameters.json* parameters file location>
    ```

    > **Note: Make sure that you are using the same Azure resource group name as the one you've just used in the `azuredeploy.parameters.json` file**

    For example:

    ```shell
    az group create --name Arc-AppSvc-Demo --location "East US"
    az deployment group create \
    --resource-group Arc-AppSvc-Demo \
    --name arcappsvc \
    --template-uri https://raw.githubusercontent.com/microsoft/azure_arc/main/azure_arc_app_services_jumpstart/aks/arm_template/azuredeploy.json \
    --parameters azuredeploy.parameters.json
    ```

* Once Azure resources has been provisioned, you will be able to see it in Azure portal. At this point, the resource group should have **7 various Azure resources** deployed.

    ![ARM template deployment completed](./01.png)

    ![New Azure resource group with all resources](./02.png)

## Windows Login & Post Deployment

* Now that first phase of the automation is completed, it is time to RDP to the client VM using it's public IP.

    ![Client VM public IP](./03.png)

* At first login, as mentioned in the "Automation Flow" section above, the [_AppServicesLogonScript_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/arm_template/artifacts/AppServicesLogonScript.ps1) PowerShell logon script will start it's run.

* Let the script to run its course and **do not close** the PowerShell session, this will be done for you once completed. Once the script will finish it's run, the logon script PowerShell session will be closed, the Windows wallpaper will change and the Azure web application will be deployed on the cluster and be ready to use.

    > **Note: As you will notices from the screenshots below, during the Azure Arc enabled app services environment, the _log-processor_ service pods will be restarted and will go trough multiple Kubernetes pod lifecycle stages. This is normal and can safely be ignored. To learn more about the various Azure Arc enabled app services Kubernetes components, visit the official [Azure Docs page](https://docs.microsoft.com/en-us/azure/app-service/overview-arc-integration#pods-created-by-the-app-service-extension).**

    ![PowerShell logon script run](./04.png)

    ![PowerShell logon script run](./05.png)

    ![PowerShell logon script run](./06.png)

    ![PowerShell logon script run](./07.png)

    ![PowerShell logon script run](./08.png)

    ![PowerShell logon script run](./09.png)

  Once the script finishes it's run, the logon script PowerShell session will be closed, the Windows wallpaper will change, and both the API Management gateway and the sample API will be configured on the cluster.

    ![Wallpaper change](./10.png)

* Since this scenario is deploying both the Azure API Management instance, the API Management gateway extension and configured with a sample API, you will also notice additional, newly deployed Azure resources in the resources group (at this point you should have **12 various Azure resources deployed**. The important ones to notice are:

  * **Azure Arc enabled Kubernetes cluster** - Azure Arc enabled app services are using this resource to deploy the app services [cluster extension](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-extensions), as well as using Azure Arc [Custom locations](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-custom-locations).

  * [**Azure API Management service**](https://docs.microsoft.com/en-us/azure/api-management/) - An instance of Azure API Management needs to be created with a gateway resource before you can enable the API Management gateway extension.

  ![Additional Azure resources in the resource group](./11.png)

## API Management sef-hosted gateway

In this scenario, the Azure Arc enabled API Management cluster extension was deployed and used throughout this scenario in order to deploy the self-hosted API Management gateway services infrastructure.

* In order to view cluster extensions, click on the Azure Arc enabled Kubernetes resource Extensions settings.

  ![Azure Arc enabled Kubernetes resource](./12.png)

  ![Azure Arc enabled Kubernetes cluster extensions settings](./13.png)

Deploying the API Management gateway extension to an Azure Arc enabled Kubernetes cluster creates an Azure API Management self-hosted gateway. You can verify this from the portal by going to the Resource Group and selecting the API management service.

  ![API management service](./15.png)

Select Gateways on the Deployment + infrastructure section.

  ![Self hosted Gateway](./16.png)

A self-hosted gateway should be deployed with one connected node.

  ![Connected node on self-hosted gateway](./17.png)

In this scenario, a sample Demo conference API was deployed. To view the deployed API, simply click on the self-hosted gateway resource and select on APIs.

  ![Demo Conference API](./18.png)

To demonstrate that the self-hosted gateway is processesing API requests you need to identify two elements:

* Public IP address of the self-hosted gateway, by running the command below from the client VM.

    ```powershell
    kubectl get svc -n apimgmt
    ```

  ![Self-hosted gateway public IP](./19.png)

* API management subsription key, from the Azure portal on the API Management service resource select Subscriptions under APIs and select Show/hide keys for the one with display name "Built-in all-access subscription".

  ![Self-hosted gateway subscriptions](./20.png)

  ![Subscription key](./21.png)

Once you have obtained these two parameters, replace them on the following code snippet and run it from the client VM powershell.

    ```powershell
    $publicip = <self hosted gateway public IP>
    $subscription = <self hosted gateway subscription>
    
    $url = "http://$($publicip):5000/conference/topics"
    $headers = @{
    'Ocp-Apim-Subscription-Key' = $subscription
    'Ocp-Apim-Trace' = 'true'
    }
    $i=1
    While ($i -le 10)
    {
    Invoke-RestMethod -URI $url -Headers $headers
    $i++
    }
    ```

  ![API calls test](./22.png)

From the Azure Portal on the  overview metrics of the self-hosted gateway the API requests will be shown.

  ![API requests metrics](./23.png)

## Cleanup

* If you want to delete the entire environment, simply delete the deployed resource group from the Azure portal.

  ![Delete Azure resource group](./24.png)
