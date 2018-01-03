# Connect to Dynamics 365 Web API with AAD app-based authentication

This document illustrates steps to authenticate with Dynamics 365 Web API (OData) service using an Azure Active Directory Application credentials. It is applicable to all the CRM versions 8.2 and later. Please find the steps below.

## Create an Azure AD Application

- Log into Azure portal <https://portal.azure.com>.
- Go to Azure Active Directory -> App registrations.
- Click New application registration.
- Enter below values:
  - Name: `Name of the application`
  - Type: Web app / API
  - Sign-on URL: `https://<nameofapplication>`
- Click Create.
- Note the Application ID aka Client ID.

## Enable application to access to Dynamics 365 API

- On the registered app's page, click All Settings -> Required Permissions -> Add.
- Under Select an API, select Dynamics CRM Online (Microsoft.CRM) and click Select.
- Under Select permissions, select the checkbox 'Access CRM Online as organization users' and click Select.
- Click Done.

Once done, required permissions should look like below

![Azure AD CRM Permissions](AzureAD-CRM%20Permissions.png)

## Create new application user in Dynamics 365

- Log into Dynamics 365 application.
- Go to Settings -> Security -> Security Roles.
- Create a new security role by copying an existing security role (e.g. System Administrator).
- Go to Settings -> Security -> Users.
- Select the view 'Application Users' and select New.
- On the new user page, select User form as 'Application User' if not already selected.
- Enter user details and click save. For example:
  - Application ID: `96ad00d5-0f07-4d35-bd69-82eb30798062`
  - Full Name: `CRM Application User`
  - Primary Email: `testemail@example.com`
- On user page, click 'Manage Roles' and assign custom security role to the user account.

Reference:  <https://msdn.microsoft.com/en-us/library/mt790170.aspx#bkmk_ManuallyCreateUser>

Example screenshot below.

![CRM Application User](CRM%20Application%20User.png)

## Connect to Dynamics 365 using Application account

### Using HTTP client in C# to access Dynamics 365 Web API

Below sample program connects to Dynamics 365 Web API using client ID and client secret and executes WhomAmI Web API function. The program can be extended to call other Web API functions and actions and perform CRUD operations on entity types.

```csharp
using Newtonsoft.Json.Linq;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;

namespace Crm9Apps
{
  public static class CrmWebApiHelper
  {
    // AD Authority used for login.  For example: https://login.microsoftonline.com/myadnamehere.onmicrosoft.com/oauth2/token
    static string authority = ConfigurationManager.AppSettings["ida:Authority"];

    // Application ID of this app. This is a guid you can get from the Advanced Settings of your Auth setup in the portal.
    static string clientId = ConfigurationManager.AppSettings["ida:ClientId"];

    // Key you generate in Azure Active Directory for this application.
    static string clientSecret = ConfigurationManager.AppSettings["ida:ClientSecret"];
    static string resource = ConfigurationManager.AppSettings["ida:Resource"];
    private static readonly HttpClient httpClient = new HttpClient();

    public static async Task<string> CrmAuthToken()
    {
      string accessToken = string.Empty;
      var values = new Dictionary<string, string>{
              { "grant_type", "client_credentials" },
              { "client_id", clientId },
              { "client_secret", clientSecret },
              {"resource", resource}
            };

      var content = new FormUrlEncodedContent(values);
      var response = await httpClient.PostAsync("https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47/oauth2/token", content);
      var responseString = await response.Content.ReadAsStringAsync();
      dynamic responseJsonStr = JObject.Parse(responseString);
      accessToken = responseJsonStr.access_token;
      return accessToken;
    }

    public static async Task<Guid> CrmWhoAmI()
    {
      Guid userId = Guid.Empty;
      //Default Request Headers needed to be added in the HttpClient object.
      httpClient.DefaultRequestHeaders.Add("OData-MaxVersion", "4.0");
      httpClient.DefaultRequestHeaders.Add("OData-Version", "4.0");
      httpClient.DefaultRequestHeaders.Accept.Add(new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

      // Set the Authorization header with the Access Token received specifying the credentials.
      httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", await CrmAuthToken());
      var response = await httpClient.GetAsync(resource + "/api/data/v8.2/WhoAmI");

      var responseString = await response.Content.ReadAsStringAsync();
      dynamic responseJsonStr = JObject.Parse(responseString);
      userId = responseJsonStr.UserId;
      return userId;
    }
  }
}
```

### Using Azure Logic App to access Dynamics 365 Web API

Create an 'HTTP request' action and pass below values to make a successful connection.

```json
"authentication": {
  "audience": "https://xxxxxxxxxxxx.crm.dynamics.com",
  "authority": "https://login.windows.net/",
  "clientId": "Client ID",
  "secret": "Client Secret",
  "tenant": "tenant.onmicrosoft.com",
  "type": "ActiveDirectoryOAuth"
}
```

### Benefits of using 'HTTP request' action

- The 'HTTP request' action can serve many of the functionalities offered by 'Dynamics 365' connectors.
- It can execute Web API functions and actions.
- It is possible to send multiple request by composing a Web API batch request.
- It can execute FetchXML requests, user queries and saved queries via Web API.

![CRM API Request](CRM%20API%20Request.png)

Documentation on CRM connector for Logic Apps can be found [here](https://docs.microsoft.com/en-us/azure/connectors/connectors-create-api-crmonline).

### Example Logic Apps defintions

Connect to Dynamics 365 Web API - WhoAmI request:

```js
"actions": {
  "HTTP": {
    "inputs": {
      "authentication": {
        "audience": "https://xxxx.crm.dynamics.com",
        "authority": "https://login.windows.net/",
        "clientId": "<ClientId goes here>",
        "secret": " ClientSecret goes here ",
        "tenant": "xxxxx.onmicrosoft.com",
        "type": "ActiveDirectoryOAuth"
      },
      "method": "GET",
      "uri": "https://xxxxxxxxx.api.crm.dynamics.com/api/data/v8.2/WhoAmI"
    },
    "runAfter": {},
    "type": "Http"
  }
}
```

Connect to Dynamics 365 Web API: Get all changes to an entity via Change Tracking

```js
"actions": {
  "HTTP": {
    "inputs": {
      "authentication": {
        "audience": "https://xxxx.crm.dynamics.com",
        "authority": "https://login.windows.net/",
        "clientId": "<ClientId goes here>",
        "secret": " ClientSecret goes here ",
        "tenant": "xxxxx.onmicrosoft.com",
        "type": "ActiveDirectoryOAuth"
      },
      "headers": {
        "Prefer": "odata.track-changes"
      },
      "method": "GET",
      "uri": "https://xxxxx.api.crm.dynamics.com/api/data/v8.2/phonecalls"
    },
    "runAfter": {},
    "type": "Http"
  }
}
```

Connect to Dynamics 365 Web API: Batch request

```js
"actions": {
  "HTTP": {
    "inputs": {
      "authentication": {
        "audience": "https://xxx.crm.dynamics.com",
        "authority": "https://login.windows.net/",
        "clientId": "<ClientId goes here>",
        "secret": " ClientSecret goes here ",
        "tenant": "xxxxx.onmicrosoft.com",
        "type": "ActiveDirectoryOAuth"
      },
      "body": "@join(variables('varPayload'),'\r\n')",
      "headers": {
        "Content-Type": "multipart/mixed;boundary=batch_AAA123"
      },
      "method": "POST",
      "uri": "https://xxxxxx.api.crm.dynamics.com/api/data/v8.2/$batch"
    },
    "runAfter": {
      "varPayload": [
        "Succeeded"
      ]
    },
    "type": "Http"
  },
  "varPayload": {
    "inputs": {
      "variables": [
        {
          "name": "varPayload",
          "type": "Array",
          "value": [
            "--batch_AAA123",
            "Content-Type:multipart/mixed;boundary=changeset_AAA124",
            "",
            "--changeset_AAA124",
            "Content-Type:application/http",
            "Content-Transfer-Encoding:binary",
            "Content-ID: 1",
            "",
            "POST https://xxxxx.api.crm.dynamics.com/api/data/v8.2/tasks HTTP/1.1",
            "Content-Type:application/json;type=entry",
            "",
            "{\"subject\":\"Task 11 in batch\"}",
            "--changeset_AAA124",
            "Content-Type:application/http",
            "Content-Transfer-Encoding:binary",
            "Content-ID: 2",
            "",
            "POST https://xxxxxx.api.crm.dynamics.com/api/data/v8.2/tasks HTTP/1.1",
            "Content-Type:application/json;type=entry",
            "",
            "{\"subject\":\"Task 12 in batch\"}",
            "--changeset_AAA124",
            "Content-Type:application/http",
            "Content-Transfer-Encoding:binary",
            "Content-ID: 3",
            "",
            "POST https://xxxxxxx.api.crm.dynamics.com/api/data/v8.2/tasks HTTP/1.1",
            "Content-Type:application/json;type=entry",
            "",
            "{\"subject\":\"Task 13 in batch\"}",
            "--changeset_AAA124--",
            "",
            "--batch_AAA123"
          ]
        }
      ]
    },
    "runAfter": {},
    "type": "InitializeVariable"
  }
}
```
