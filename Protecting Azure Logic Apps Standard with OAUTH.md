# Protecting Azure Logic Apps Standard with OAUTH

## Summary

Protecting services with OAUTH is commonplace across Azure, with out of the box support for services such as Azure Logic Apps, Azure Functions, Azure App Service and Azure API Management.  The Consumption tier of Logic Apps has had OAUTH on HTTP requests for a while now (see [Logic Apps Authorisation](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#enable-oauth)), and a similar feature is now available in the Standard tier of Logic Apps.

Logic Apps Standard is built on top of Azure Functions, so has access to the Easy Auth feature available to both Azure Functions and Azure App Service. At the time of writing the portal experience isn't avaialble, but can be enabled through the REST API. This blog builds on the article published [here](https://techcommunity.microsoft.com/t5/integrations-on-azure-blog/trigger-workflows-in-standard-logic-apps-with-easy-auth/ba-p/3207378#:~:text=Sometimes%20called%20%22Easy%20Auth%22%2C,workflow%20invocations%20possible%20through%20triggers.)

## Scenarios

We will cover two examples where we protect the Logic App workflow with Azure AD and authorise based on audience and role:

* Call the Logic App from Postman where we fetch a  token from Azure AD with a client id and secret
* Call the Logic App from Azure API Management using Managed Identity

We will also show how to check for specific roles within the Logic App, allowing for different workflows to have different authorisation rules within the same Logic App.

## Azure AD Configuration

The security context in Azure AD is configured through an App Registration. There is one App Registration required
to represent the Logic App workflow (the service), then we will create additional App Registration(s) to represent clients that want to call the workflow. We create roles on the service app registration that can be used to assign to a client (whether managed identity or client id/secret). Easy Auth will be configured to check the audience, then the workflow can make further checks against roles or scopes.

Create Logic App Application Registration

![Register Application
](Register%20Backend%20App%20Registration%20for%20Logic%20App.png)

Each Application Registration has an Application ID Uri, which by default is a GUID value. It's best to change this to something more meaningful so when validating the JWT it will be clear on the permission being checked. To set the Application ID Uri click on Application ID Uri on the overview page of the Application Registration:
![Set Application ID Uri](Application%20ID%20URI.png)

Configure the Application ID Uri to something meaningful, bearing in mind there are strict rules on what this value can be, for example if you use a domain it must be one you own and configured within Azure AD.

![Application ID Uri](Application%20ID%20URI%20Set.png)

## Create Application Roles

We will create a role within the Application Registration, in this case a role to allow a client application to read Contoso orders. First, navigate to Roles, then click on Create App Role, as follows:
![](Create%20Application%20Role.png)


The following page is displayed allowing a role to be created. In our case we are creating a role called Read Contoso Orders:
![Create Role](Create%20Role.png)

## Create Client Application

Next, we need to create a client Application Registration then assign permission for this application to have access to the Read Contoso Orders role.

Within the Consto Client Application Registration, navigate to API Permissions, then click "Add a Permission" where the following screen is displayed:

![Request Permissions](Request%20API%20Permissions.png)

Within "APIs my organization uses", enter the name of the backend Application Registration, in this case ContosoBackendWorkflow. When it appears in the search results click it, which will display Application Permissions sceen. Select Application Permissions, then select the ReadContosoOrders role and click "Add Permissions", as follows:
![Assign Backend Permissions](Assign%20backend%20role%20permissions.png)

You should see the permission listed against the Contoso Client Application Registration:
![View Application Permissions](Contoso%20Client%20Permissions%20View.png)

Although the permission is listed, permission has not yet been granted. To do this, login as an Azure AD Administrator and click "Grant admin consent". For automated scenarios, or where Azure AD Admin is not appropriate, see [here](https://learn.microsoft.com/en-us/graph/permissions-grant-via-msgraph?pivots=grant-application-permissions&tabs=http) for details on how to automate this without Admin consent. Once permission has been granted, there should be a green tick to indicate the permission:

![Permission Granted](Client%20Permission%20Granted.png)

## Logic App Configuration

We now need to create a Logic App in the Standard tier, configure Easy Auth, then validate the Audience and role within the JWT the client will send in the Authorisation header.

To create a Standard tier Logic App workflow with an HTTP trigger. Details on how to do this can be found [here](https://learn.microsoft.com/en-us/azure/logic-apps/create-single-tenant-workflows-azure-portal).

Once the Logic App has been created, configure Easy Auth for the Logic App. At the time of writing there is no support in the Azure Portal for this, but we can use the REST API. 

[](Azure%20AD%20Auth%20Logic%20App%20Config%20id.png)
The following values need to be set, highlighted  below:

issuerUrl - https://login.microsoftonline.com/{your tenant id}/v2.0

clientId - the client id of the first Application Registration we created (this can be retrieved from the overview page of the App Registration)


