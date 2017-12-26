# CrmWebApiOAuth

Authenticate Dynamics 365 oData /Web API using an Azure Active Directory Application Client ID / Client secret.

## Introducing the Azure AD Application Identity for Dynamics 365 Web API

Microsoft introduced the ability to authenticate Dynamics 365 oData /Web API using an Azure Active Directory Application (Client ID/Client Secret). Azure AD application together with a client secret, can replace username and password authentication in many scenarios. This is applicable to all the CRM versions 8.2 and later.

In a nutshell, it mitigates the risk of having username/password readable and thus less safe.
For example, it could look like below:

    a.Username: username@domain.com
    b.Password: Password

On the other hand, Client ID /Client Secret is difficult to comprehend making it safer.
For example, it could look something like below

    a.Client ID: 82068d67-54a7-4698-a4eb-876dcc70b3c1
    b.Client Secret: BD|m_hIy461-E!p&;u0l@7sPCab?579LA`iP5ek|5rD]V

We can perform all Dynamics 365 Web API actions using Client ID/ Client Secret instead of using the traditional username/password credentials.
This document provides step by step instructions to create and leverage this feature.

## Create an Azure AD Application

    1.Log into Azure portal (https://portal.azure.com)
    2.Go to Azure Active Directory -> App registrations
    3.Click New application registration
    4.Enter the below values
        a.Name: Name of the application
        b.Type: Web app / API
        c.Sign-on URL: Ex: https://Nameofapplication
    5.Click Create
    6.Note Client ID aka Application ID
    7.Assign Dynamics CRM Online API access
        a.Click All Settings -> Required Permissions
        b.Click Add -> Select an API
        c.Select Dynamics CRM Online (Microsoft.CRM)
        d.Click Select
        e.Click Select permissions
        f.Select the checkbox ‘Access CRM Online as organization users’
        g.Click Select
        h.Click Done
        i.Assign Dynamics CRM Online API access rights
    8.Once done, required permissions should look like below

![alt text](https://github.com/anilvem1/CrmWebApiOAuth/blob/master/AzureAD-CRM%20Permissions.png)

## Create new Application User in Dynamics 365

    1.Log into Dynamics 365 CRM application
        a.Pre-requisite: CRM version 8.2 and later
    2.Go to Settings -> Security Roles
    3.Create a new security role by copying an existing OOB security role (Ex: System Administrator)
    4.Go to Settings -> Security -> Users
    5.Select the view ‘Application Users’
    6.Create CRM user via Security -> Users -> Application Users
    7.Click New
    8.Enter User Name as Client ID
    9.Enter Full Name & Primary Email of your choice
        a.Ex: Full Name: CRM Application User & Primary Email: testemail@xxxxx.com
    10.Click Save
    11.Assign the above custom security role to this user account
    12.Reference:  https://msdn.microsoft.com/en-us/library/mt790170.aspx#bkmk_ManuallyCreateUser
    13.Example screenshot below

![alt text](https://github.com/anilvem1/CrmWebApiOAuth/blob/master/CRM%20Application%20User.png)

## Connect to Dynamics 365 CRM using Application user account

    1.C# code:
        a.Connect to Dynamics 365 CRM Web API using client ID/ client secret
        b.Perform Web API operations such as CRUD, Execute and others
        c.Please check in appendix for code sample
    2.Logic App definition:
    3.Using HTTP request type action, we should be able to make a connection to CRM Web API
        a.Here we need to pass the below values to make a successful connection
            "authentication": {
            "audience": "https://xxxxxxxxxxxx.crm.dynamics.com",
            "authority": "https://login.windows.net/",
            "clientId": "Client ID",
            "secret": "Client Secret",
            "tenant": "tenant.onmicrosoft.com",
            "type": "ActiveDirectoryOAuth"
            }
    4.This can replace the existing CRM connectors available

![alt text](https://github.com/anilvem1/CrmWebApiOAuth/blob/master/CRM%20API%20Request.png)

    5.Can execute user / saved queries via Web API in a single HTTP request
    6.Can execute Web API batch / bulk operations are supported (# actions are reduced)
    7.Can execute unbound / bound custom actions via Web API in a single HTTP request
    8.Can execute Saved / user query (FetchXML) via Web API in a single HTTP request
        a. https://xxxxxxxxx.api.crm.dynamics.com/api/data/v8.2/accounts?userQuery=3BB1D60D-D4E1-E711-80FF-3863BB2E0320
    9.Can impersonate with another user if Azure AD App user is not an option while doing CRUD operations in Dynamics 365 CRM
    10.Can also replace triggers when a CRM record is created / updated using Change tracking feature of Dynamics 365 CRM
    11.Almost all Web API operations can be executed in Logic App using this approach without writing any piece of code

## Observations / Benefits

    1.No dependency on username / password. More secure with Azure AD client ID / client secret to connect to Dynamics 365 CRM
    2.Logic App limitations with connector can be mitigated (300 actions per minute). Using this HTTP request approach, we don’t have any limit on number of actions per minute
    3.Created a Logic App with 10 CRM actions in it and set the recurrence to run every 1 second, 620 actions per minute successful. No errors

![alt text](https://github.com/anilvem1/CrmWebApiOAuth/blob/master/CRM%20API%20LA%20Run.png)

        a.Compared to CRM connector, all available CRM Web API functionality can be used within Logic Apps / Flow
        b.Ex: WhoAmI, any functions / actions
        c.Batch / bulk operations
    5.Any C# code can use this approach of connecting to CRM using Azure AD App credentials (Ex: Azure Function App / Custom APIs)

## Appendix

    1)CRM connector for Logic Apps can be found here: https://docs.microsoft.com/en-us/azure/connectors/connectors-create-api-crmonline
    2)Logic Apps examples can be found below
    3)Connect to CRM Web API – WhoAmI request:

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
    4)Connect to CRM Web API: Get all changes to an entity via Change Tracking

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
                                    "runAfter": {
                                    },
                                    "type": "Http"
                                }
                    }

    5)Connect to CRM Web API: Batch request

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

    6)Sampe C# code to connect to Dynamics 365 CRM
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
                    static string authority = ConfigurationManager.AppSettings["ida:Authority"];  // the AD Authority used for login.  For example: https://login.microsoftonline.com/myadnamehere.onmicrosoft.com/oauth2/token
                    static string clientId = ConfigurationManager.AppSettings["ida:ClientId"]; // the Application ID of this app.  This is a guid you can get from the Advanced Settings of your Auth setup in the portal
                    static string clientSecret = ConfigurationManager.AppSettings["ida:ClientSecret"]; // the key you generate in Azure Active Directory for this application
                    static string resource = ConfigurationManager.AppSettings["ida:Resource"]; // Ex: https://xxxxxxxxxxx.crm.dynamics.com
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
                        //Default Request Headers needed to be added in the HttpClient Object
                        httpClient.DefaultRequestHeaders.Add("OData-MaxVersion", "4.0");
                        httpClient.DefaultRequestHeaders.Add("OData-Version", "4.0");
                        httpClient.DefaultRequestHeaders.Accept.Add(new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

                        //Set the Authorization header with the Access Token received specifying the Credentials
                        httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", await CrmAuthToken());
                        var response = await httpClient.GetAsync(resource+ "/api/data/v8.2/WhoAmI");

                        var responseString = await response.Content.ReadAsStringAsync();
                        dynamic responseJsonStr = JObject.Parse(responseString);
                        userId = responseJsonStr.UserId;
                        return userId;
                    }
                }
            }
