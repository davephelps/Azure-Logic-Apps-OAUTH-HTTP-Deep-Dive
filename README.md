# Deep Dive on Protecting Azure Logic Apps Standard with Microsoft Entra ID (OAUTH) and calling from Azure API Management

## Summary 

Protecting services with Microsoft Entra ID (OAUTH) is commonplace across Azure, with out of the box support for services such as Azure Logic Apps, Azure Functions, Azure App Service and Azure API Management.  The Consumption tier of Logic Apps has had OAUTH for HTTP requests for a while now (see [Logic Apps Authorisation](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#enable-oauth)), and a similar feature is now available in the Standard tier of Logic Apps.

Logic Apps Standard is built on top of Azure Functions, so has access to the *Easy Auth* feature available to both Azure Functions and Azure App Service. At the time of writing the portal experience isn't avaialble, but can be enabled through the REST API. This blog builds on the article published [here](https://techcommunity.microsoft.com/t5/integrations-on-azure-blog/trigger-workflows-in-standard-logic-apps-with-easy-auth/ba-p/3207378#:~:text=Sometimes%20called%20%22Easy%20Auth%22%2C,workflow%20invocations%20possible%20through%20triggers.)

## Scenarios

We will cover three ways to authorise a client from a Logic App:

* Authorise based on *audience* contained in the JWT
* Authorise based on *audience* and the unique identifier (Application ID) of the client contained in the JWT
* Authorise based on *audience* and *roles* (part of the JWT) assigned to the calling application

We will talk through these scenarios in detail later. We will also test a Microsoft Entra ID protected Logic App in two ways:

* Call the Logic App from Postman where a token is fetched from Microsoft Entra ID with a Application ID and secret
* Call the Logic App from Azure API Management using *Managed Identity*

# Logic App Microsoft Entra ID Configuration

To enable OAUTH we need to create and configure an *Application Registration* in Microsoft Entra ID. The Application Registration is the security context in Microsoft Entra ID representing the Logic App (the service). We will then create additional App Registrations or Managed Identities to represent clients. These clients will be used to obtain access tokens to to call the Logic App workflow.

## Create Logic App Application Registration

As shown in the screenshot below, create a new Application Registration, provide a name but leave everything else as default. In our case, the App Registration is called "ContosoBackendWorkflow":

![Register Application
](Register%20Backend%20App%20Registration%20for%20Logic%20App.png)

## Configure Application ID Uri
Each Application Registration has an *Application ID Uri*, which by default is a GUID value. This value becomes the *audience* in the JWT token. It's best to change this to something more meaningful so when validating the JWT it will be clear on the permission being checked. There are strict rules on the value of *Application ID Uri*, for example if you use a domain it must be verified. See [this link](https://learn.microsoft.com/en-us/entra/identity-platform/security-best-practices-for-app-registration#application-id-uri) for more details. To set the Application ID Uri click on Application ID Uri on the overview page of the Application Registration:

![Set Application ID Uri](Application%20ID%20URI.png)



![Application ID Uri](Application%20ID%20URI%20Set.png)



## Logic App Configuration

We now need to create a Logic App in the Standard tier, configure Easy Auth, then validate the *audience*.

Although we can run and test Logic Apps locally using Visual Studio Code, we will create the Logic App in Azure. Create a simple HTTP request/response Standard tier Logic App workflow, details on how to do this can be found [here](https://learn.microsoft.com/en-us/azure/logic-apps/create-single-tenant-workflows-azure-portal).

Once the Logic App has been created, configure Easy Auth for the Logic App. At the time of writing there is no support in the Azure Portal for this, but we can use the Azure Management REST API or Bicep. We will be using the REST API.

A sample payload we can use as a template can be found [here](<Logic App OAUTH Config.json>). Edit the file locally, substituing the following values:

| Name | Description |
| -----| ------------|
| {{subscription_id}} | subscription identifier where the Logic App is deployed |
| {{resource_group_name}} | resource group where the Logic App is deployed |
| {{logic_app_name}} | name of the Logic App where the Easy Auth config is to be applied |
| {{region}} | region where the Logic App is deployed |
| {{tenantid}} | tenant id (visible on the overview page of Microsoft Entra ID) where the logic App is deployed |
| {{issuerUrl}} | https://login.microsoftonline.com/{your tenant id}/v2.0 |
| {{la_client_id}} | the Application ID of Application Registration created that represents the Logic App|
| {{la_audience}} | the audience of the Application Registration (i.e., the Application Id URI)
||


Once the file has been saved we will use the Azure CLI to deploy the changes. Azure CLI can be used in a variety of ways, including from Visual Studio, Visual Studio Code and the Azure Portal CLI. We will use Visual Studio.

Open Visual Studio and start a new Terminal by selecting *Terminal* from the *View* menu.

When the terminal has loaded, login to Azure typing *az login* and enter your Azure credentials. Enter the following command, substituting with your values as described above.

*az rest* is a command that is part of the Azure CLI, that once logged in, will automatically add the authorization header making it easier to call the Azure Management API to manage Azure Services.

```
az rest --method put --uri https://management.azure.com/subscriptions/{{subscription_id}}/resourcegroups/{{resource_group_name}}/providers/Microsoft.Web/sites/{{logic_app_name}}/config/authsettingsV2?api-version=2021-02-01 --body '@Logic App OAUTH Config.json'
```

### Validate Security Configuration with PostMan

To show the Logic App is now protected with Microsoft Entra Id, test the workflow in Azure by calling it through PostMan. 

Create a new API request in PostMan called "Call Logic App", paste in the HTTP trigger URL (obtained from the overview page of the Logic App workflow created earlier) and press *Send*.

The call should fail with a security error as we have not yet obtained a token from Microsoft Entra Id to pass to the workflow.

### Test Locally using Microsoft Entra Id with PostMan
To test the workflow, a valid token must be obtained from Microsoft Entra ID which is then passed to the workflow (from PostMan) as an HTTP header.

Create another App Registration called "Contoso Sales Client". We will use this App Registration represent a test client to call the workflow. Create as single tenant and **make a note of the Application (client) ID on the Overview page as it will be required later**. "Application ID" and "Client ID" are the same thing and can be used interchangeably.

Click the *Certificates & Secrets* link, and click *New Client Secret* where a secret can be created. **Make a note of the secret as it is only displayed once and will be required later**.

![Client Secret](<Client - secret.png>)

### Obtain a Token from PostMan

To obtain a token from PostMan, create a new API Request based on an HTTP POST called "Get Logic App Token":

Note: Postman has a feature called *Environments*, which allow variables to be configured and referenced in API requests. Variables are referenced using double brace notation, for example {{tenant_id}}. Either create a new environment and configure the values below, or enter the real values into the API call. It's generally best practice to use environments as they can be re-used across more than one API call.

Note: Postman has a feature to automate fetching a token, available in the Authorization tab. We are not using that feature but setting the values in the HTTP request to demonstrate the mechanics of the authorisation request.

Set the following in an *x-www-form-urlencoded* body: 

- Set the URL to *https://login.microsoftonline.com/{{tenantid}}/oauth2/v2.0/token* by changing **{{tenantid}}** to the ID of your tenant. This can be found by navigating to Microsoft Entra Id and clicking the **Overview** page and copying **Tenant ID**:

- client_id - this is the Application ID copied from the Test Client Application Registration just created
- client_secret - this is the secret created earlier from the Test Client Application Registration just created
- grant_type - this is set to *client_credentials* and represents the OAUTH2 flow for one system talking to another (i.e., no user present) 
- scope - this is the Application ID URI created in the Logic App App Registration with **/.default** appended. For example, https://yourtenant.onmicrosoft.com/contoso/app/backend/.default

For example:

![Postman Get Token](<Postman - Get Token.png>)



Click **Send** and the token should be returned:

The token is encoded, so the easiest way to view it is to open a browser and navigate to [https://jwt.ms](https://jwt.ms), then paste the token content (i.e., the access_token part of the response) into the website. The token should then be decoded showing all the claims in the JWT. Note the **audience** which is the value configured against the Logic App App Registration (ContosoBackendWorkflow). Also note the **appid** which is the Application ID of the *client* App Registration.

The *audience* value in the JWT will be checked at runtime by the EasyAuth configuration we created earlier. Only if the client has a token with this specific audience will the request be allowed.

### Test the Logic App from PostMan using a Microsoft Entra Id Token

The next step is to test Logic App workflow by passing a valid JWT token to the Logic App. We can do this from PostMan in one of two ways:
- copy the access token returned above and paste it into the Authorization HTTP header
- update the PostMan call where the token is retrieved to automatically update a variable to negate the need to copy and paste

We will use the second option. Navigate to the PostMan API request called "Get Logic App Token", then click on the *Tests* tab and enter the following code:

```
    var json = JSON.parse(responseBody);
    postman.setEnvironmentVariable("la_bearer", "Bearer " + json.access_token);
```
This will update a variable called *la_bearer* with the access token allowing us to then reference it when calling the Logic App. The token has a lifetime, so after a period of time it will expire and a new token will need to be retrieved.

Next, navigate to the API request called "Call Logic App", then to *Headers* and add a header called "Authorization" with a value of "{{la_bearer}}", for example:

![Call Logic App](<Postman - Call Logic App.png>)

Click *Send* and the Logic App should return with a 200 OK response, indicating the token is valid and has the correct audience. If the call fails, check the token in https://jwt.ms to ensure the token is present. Also make sure the Logic App Easy Auth configuration is checking for the same audience.

## Authorising Clients
As mentioned at the start, there are three ways for the Logic App to authorise clients:

* Authorise based on *audience* contained in the JWT
* Authorise based on *audience* and the unique identifier (Application ID) of the client contained in the JWT
* Authorise based on *audience* and *roles* (part of the JWT) assigned to the calling application

So far, we have implemented the first option - to authorise based on the *audience*. You may be wondering what's to stop *anyone* from creating a JWT token such as the one returned from Microsoft Entra Id and just passing it to the Logic App. When a token is returned from Microsoft Entra Id it is *digitally signed* by Microsoft Entra Id using a private certificate that only Microsoft Entra Id has. The first step the authorisation framework takes is to validate the signature of the JWT token using the Microsoft Entra Id public key. This allows the runtime to verify the token hasn't been tampered with or created with a different key.

### Authorise based on the unique identifier (Application ID)
By default however, *any* client within the tenant is able to request a token from Microsoft Entra ID. While this may be the desired behaviour, a more likely scenario is to only allow *specific* clients to call the Logic App.

We can update the Logic App configuration to specify a list of Application ID values so only *those* applications can access the Logic App. There is a second json template, [Client Auth](<Logic App OAUTH Config_client_auth.json>) which contains a section called *defaultAuthorizationPolicy* which has two json objects, *allowedPrincipals* and *allowedApplications*, shown below:

```
    "defaultAuthorizationPolicy": {
        "allowedPrincipals": {
            "identities": [
                "{object id to restrict}",
                "{object id to restrict}"
                ]
        },
        "allowedApplications": [
            "{Application ID to restrict}",
            "{Application ID to restrict}"
        ]
    }
```
 Both options allow restriction of the calling client, either by using the Application ID or the object id, and both can be an array to restrict multiple clients. If the *allowedPrincipals* element is present, this must be used and will override anything in the *allowedApplications* object, so use one or the other, not both. In our case we will add the Application Id used to test from PostMan.


Navigate to the Postman API Request called "Get Logic App Token", copy the client_id value and paste it into the *allowedApplications* array. Remove the *allowedPrincipals* object entirely, or just the *identities* section.

The configuration should look similar to the example below:
```
    "defaultAuthorizationPolicy": {
        "allowedApplications": [
            "f3280e6a-6625-4380-2764-ad3301810ae4",
        ]
    }
```
Update the Logic App Config using the az rest command and test the Logic App.

### Authorise Based on Roles
While authorising based on the Application ID of the calling application provides fine grained control, it does mean however that the Logic App configuration must be updated for each new client that wants to call the Logic App.

Another option that avoids the need to authorise specific clients is to use *roles*. With roles, we can create a role (or series of roles) on the Logic App Application Registration, then assign roles to clients. The role will only be contained in the JWT token if the client has specifically been granted access. We can then disable the feature that allows *any* client within the tenant to obtain a token. To do this, navigate to the Logic App Application Registration (ContosoBackendWorkflow) and click on *Managed Application in Local Directory*. Once the Enterprise Application is visible, click on *Properties* and change *Assignment Required* to *Yes* as follows:

![Assignment Required](<Enterprise Application - Assignment required.png>)

Once Assignment Required has been set to Yes, our Logic App will no longer succeed. This is because we now need to assign a role to the client. Doing this will set the audience and role in the JWT token.

## Creating Roles
We will create a role within the Application Registration, in this case a role to allow a client application to read Contoso orders. First, navigate to *Roles*, then click *Create App Role*, as follows:

![Create Role](Create%20Application%20Role.png)


The following page is displayed allowing a role to be created. In our case we are creating a role called *Read Contoso Orders*:

![Create Role](Create%20Role.png)

## Create Client Application

Next, we need to assign permission for the client application to have access to the Read Contoso Orders role. The client application could be an App Registration with client id and secret (to test from Postman) or Managed Identity assigned to an Azure Service. We will test from Postman first, so navigate to the "Contoso Sales Client" App Registration and click *API Permissions*, then click "Add a Permission" where the following screen is displayed:

![Request Permissions](Request%20API%20Permissions.png)

Within "APIs my organization uses", enter the name of the Logic App Application Registration, in this case ContosoBackendWorkflow. Select ContosoBackendWorkflow from the search results to display the Application Permissions screen. Select Application Permissions, then select the *ReadContosoOrders* role and click *Add Permissions*, as follows:

![Assign Backend Permissions](Assign%20backend%20role%20permissions.png)

You should see the permission listed against the Contoso Client Application Registration:

![View Application Permissions](Contoso%20Client%20Permissions%20View.png)

Although the permission is listed, permission has not yet been granted. To do this, login as an Microsoft Entra ID Administrator and click "Grant admin consent". For automated scenarios, or where Microsoft Entra ID Admin is not appropriate, see [here](https://learn.microsoft.com/en-us/graph/permissions-grant-via-msgraph?pivots=grant-application-permissions&tabs=http) for details on how to automate this without full Admin consent. Once permission has been granted, there should be a green tick:

![Permission Granted](Client%20Permission%20Granted.png)

## Test from PostMan
The final step is to request a token and verify the audience and role have been set correctly in the JWT token. In PostMan, navigate to the "Get Logic Apps Token" API Request and press *Send*. If the request is successful (which it only will be if the permission has been granted), copy the access token and view in https://jwt.ms. There should be an entry called *roles* with an entry for the role assigned above.

## Fine Grained Authorisation from Logic Apps Workflows
A further step we can take is to check the role within the Logic App Workflow itself. This allows for fine grained authorisation, for example two roles could be created, one to read orders and one to create orders. Two workflows in the same Logic App could then check for a specific role, which can easily be achieved using expressions in Logic Apps. For example, in the following example we are using a *Compose* action to retrieve the role, then a *Condition* to check if the role was present or not:

![Check Claims](<Logic Apps - Check Claims.png>)

The workflow json snippet is as follows:
```
    "Check_Claims": {
        "inputs": {
            "from": "@json(base64ToString(triggerOutputs()['headers']['X-MS-CLIENT-PRINCIPAL']))?['claims']",
            "where": "@and(equals(item()?['typ'], 'roles'),equals(item()?['val'], 'sales.write'))"
        },
        "runAfter": {
            "Response": [
                "SUCCEEDED"
            ]
        },
        "type": "Query"
    },
    "Authorised": {
        "actions": {
            "Success": {
                "inputs": {
                    "statusCode": 200
                },
                "kind": "http",
                "type": "Response"
            }
        },
        "else": {
            "actions": {
                "Unauthorised": {
                    "inputs": {
                        "statusCode": 401
                    },
                    "kind": "http",
                    "type": "Response"
                }
            }
        },
        "expression": {
            "and": [
                {
                    "equals": [
                        "@length(body('Check_Claims'))",
                        1
                    ]
                }
            ]
        },
        "runAfter": {
            "Check_Claims": [
                "Succeeded"
            ]
        },
        "type": "If"
    }
```
# Calling Logic Apps from Azure API Management
So far, we have only tested calling the Logic App from PostMan, but calling from another service is very straightforward. To call from Azure API Management for example, first we need to add Managed Identity (either System or User Assigned), then create an API to call the Logic App workflow created above. 

### Configure Managed Identity in Azure API Management
Managed Identity settings in Azure API Mnagement can be found by navigating to *Security* then *Managed Identities* where System and User Assigned Managed Identities can be configured. Here is an example of a system assigned managed identity:

![System Assigned](<APIM System Assigned MI.png>)


### Configure API Management Policy for Managed Identity
The only additional step we need to do is to add a *policy* to the API to fetch a token using the configured managed identity. For example the following is using System Assigned Managed Identity:

```
<policies>
    <inbound>
        <base />
        <authentication-managed-identity resource="https://dptestadi.com/contoso/app/backend" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

To use User Assigned Managed Identity, navigate to the User Assigned Managed Identity, copy the Application ID and use the following policy:
```
<policies>
    <inbound>
        <base />
        <authentication-managed-identity resource="https://dptestadi.com/contoso/app/backend" client-id="your managed identity Client ID/Application ID" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

Azure API Management will manage fetching a token using managed identity without the need to manage Client ID and secret, token expiry etc.

## Authorise an API Management Request From Logic Apps using Audience and Application ID
When making a call from API Management to our Microsoft Entra ID protected Logic App, we authorise using the same techniques already discussed by configuring *Managed Identity* in API Management. Once this has been done, we can either check for the Application ID of the Managed Identity through configuration of Easy Auth, or we can use the *roles* approach and simply assign a role to the Managed Identity. Only if the role is assigned with the request be authorised.

### System Assigned Managed Identity
For User Assigned Managed Identity, navigate to *Identities* in API Management and view the *System Assigned* tab. The Object ID displayed is *not* the Application ID, but the underlying object identifier of the Managed Identity. The object id can be used, but would need to be included in the *allowedPrincipals* section of the EasyAuth configuration. To find the Application ID instead, copy the Object ID and search for it within Microsoft Entra ID. Click on the *Enterprise Application* in the search results that is the same name as the Azure API Management instance. From here, the Application ID is visible where it can then be used in the *allowedApplications* section of the Easy Auth Configuration.

### User Assigned Managed Identity
For User Assigned Managed Identity, navigate to *Identities* in API Management and view the *User Assigned* tab. There may be more than one managed identity displayed - click the identity of interest which will navigate to the identity configuration. From here, note the Application ID where it can then be used in the *allowedApplications* section of the Easy Auth Configuration.

### Authorise API Management using Roles
To grant roles to the Azure API Management Managed Identity, we need to find the Application ID, then grant permission using either Powershell or the REST API. It is not possible to do this through the Application Registration user interface as discussed earlier.

To grant permission using PowerShell, we can use the following script (bear in mind the comments about permissions required to grant Application permissions, discussed above):

```
# App Registration for Logic App
$serviceAppName="ContosoBackendWorkflow"
# role to assign
$serviceAppRoleName="Create Contoso Orders"

# APIM Service Principal (managed identity object id)
$apimManagedIdentityObjectId = "f88c2d51..."
$serviceSP = Get-AzureADServicePrincipal -Filter "displayName eq '$serviceAppName'"

# Find the Service Application Role
$appRole = $serviceSP.appRoles | Where-Object { $_.DisplayName -eq $serviceAppRoleName }

# Now apply the permissions to the client
New-AzureADServiceAppRoleAssignment -ObjectId $apimManagedIdentityObjectId -PrincipalId $apimManagedIdentityObjectId -Id $appRole.Id -ResourceId $serviceSP.ObjectId

```
Now, when API Management requests a token, the role will be included in the JWT token which can be checked by the Logic App Workflow.