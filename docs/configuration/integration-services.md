# Integration Services

## Overview

Integration Services acts as the entry point for REST API requests and
routes them to the appropriate SCExpert Connect plugin.

### Overall Flow

``` text
Client / Postman
      |
      v
Integration Services API
      |
      | Warehouse + BOType
      v
IntegrationServicesPluginRegistry
      |
      | IP + Port + URLPath
      v
Matching SCExpert Connect Plugin
      |
      v
WMS / Client Database
```

------------------------------------------------------------------------

## 1. Source Code

For **SCExpert 4.12.1**, the Integration Services code is available in
Azure DevOps under:

``` text
Repository:
SCExpert_WMSRestAPI_4.12.1
```

![Integration Services
Repository](images/integration-services/01-repository.png)

The repository contains:

``` text
SAAS.ConnectPlugins.sln
SAAS.IntegrationServices.sln
SAAS.Logic.sln
```

For Integration Services, open:

``` text
SAAS.IntegrationServices.sln
```

![Integration Services
Solution](images/integration-services/02-solution.png)

The main project is:

``` text
SAAS.IntegrationServices
```

![Integration Services
Project](images/integration-services/03-project-structure.png)

Two important controllers are:

-   `AuthenticateController.cs` -- handles API authentication.
-   `SAASInterfacesController.cs` -- handles Import API requests.

The main Import route is:

``` text
POST /IntegrationServices/api/Import/{boType}
```

------------------------------------------------------------------------

## 2. Find the Integration Services Database

On the client/Azure server, open:

``` text
Made4Net.ConnRegistration
```

![ConnRegistration](images/integration-services/04-connregistration.png)

In **Connection Manager**, the `Default` connection is normally used by
the workstation/RDT.

Change **App Name** to:

``` text
IntegrationServices
```

![Integration Services
Connection](images/integration-services/06-integration-services-connection.png)

This shows which database Integration Services is using.

In this example:

``` text
SCEWEBINTERFACESYS
```

------------------------------------------------------------------------

## 3. Plugin Routing Configuration

In the Integration Services database, run:

``` sql
SELECT *
FROM IntegrationServicesPluginRegistry;
```

![Integration Services Plugin
Registry](images/integration-services/07-plugin-registry.png)

This table tells Integration Services where each request needs to be
routed.

  Field         Purpose
  ------------- --------------------------
  `Warehouse`   Warehouse
  `BOType`      Business Object/API type
  `IP`          Plugin server IP
  `Port`        Plugin port
  `URLPath`     Plugin endpoint
  `DBName`      Client database

The actual REST API plugins can be checked using:

``` sql
SELECT *
FROM SCEXPERTCONNECTPLUGINS;
```

![SCExpert Connect
Plugins](images/integration-services/08-connect-plugins.png)

Examples include:

``` text
HTTPListener COMPANY API
HTTPListener INBOUND API
HTTPListener OUTBOUND API
HTTPListener RECEIPT API
HTTPListener SKU API
```

### How BOType Routing Works

For example:

``` text
POST http://10.7.2.108/IntegrationServices/api/Import/OUTBOUND
```

The last value, `OUTBOUND`, is the **BOType** and must match the BOType
configured in `IntegrationServicesPluginRegistry`.

Integration Services then uses the configured **IP, Port and URLPath**
to route the request to the corresponding plugin.

``` text
/api/Import/OUTBOUND
        |
        v
BOType = OUTBOUND
        |
        v
IntegrationServicesPluginRegistry
        |
        | Find matching Warehouse + BOType
        v
IP + Port + URLPath
        |
        v
SCEXPERTCONNECTPLUGINS
        |
        v
HTTPListener OUTBOUND API
```

![Postman Import
URL](images/integration-services/09-postman-import-url.png)

------------------------------------------------------------------------

## 4. Authentication Configuration

Before testing an API, obtain an authentication token.

The Integration Services username and password are stored in the
`SYS_PARAM` table of the Integration Services database.

``` sql
SELECT *
FROM SYS_PARAM
WHERE PARAM_NAME LIKE '%SAAS%';
```

![Authentication
Parameters](images/integration-services/10-sys-param-authentication.png)

Important parameters:

-   `SAASAPIUserName` -- Login API username
-   `SAASAPIPassword` -- Login API password
-   `SAASApiAuthTokenExp` -- Token expiry configuration

!!! warning "Security" Do not store actual passwords or AuthTokens in
documentation or shared screenshots.

------------------------------------------------------------------------

## 5. Get AuthToken Using Postman

First call the Login API:

``` text
POST http://<Integration-Services-IP>/IntegrationServices/api/login
```

Example:

``` text
POST http://10.7.2.108/IntegrationServices/api/login
```

In **Authorization**, select **Basic Auth** and use the username and
password configured in `SYS_PARAM`.

![Login Basic
Authentication](images/integration-services/11-login-basic-auth.png)

Send the request.

A successful request returns:

``` text
200 OK
"Authorized"
```

![Login Authorized](images/integration-services/12-login-authorized.png)

Open the response **Headers** and copy the `AuthToken`.

![AuthToken Header](images/integration-services/13-login-auth-token.png)

------------------------------------------------------------------------

## 6. Test the Import API

Open the API that needs to be tested.

Example:

``` text
POST http://10.7.2.108/IntegrationServices/api/Import/OUTBOUND
```

Go to **Authorization → API Key** and configure:

``` text
Key   = AuthToken
Value = <token copied from Login API>
```

![Import API
Authorization](images/integration-services/14-import-api-auth-token.png)

Add the required request body and send the request.

------------------------------------------------------------------------

## Complete Integration Services Flow

``` text
              SCEWEBINTERFACESYS
                     |
          +----------+-----------+
          |                      |
          v                      v
       SYS_PARAM      IntegrationServicesPluginRegistry
          |                      |
 Username + Password       BOType / IP / Port
          |                URLPath / DBName
          v                      |
      Login API                  |
          |                      |
          v                      |
      AuthToken                  |
          |                      |
          +----------+-----------+
                     |
                     v
              Import API Request
                     |
       /api/Import/{BOType}
                     |
                     v
          SAASInterfacesController
                     |
                     v
           Match Warehouse + BOType
                     |
                     v
       IntegrationServicesPluginRegistry
                     |
                     v
            IP + Port + URLPath
                     |
                     v
          SCEXPERTCONNECTPLUGINS
                     |
                     v
          Matching REST API Plugin
                     |
                     v
               WMS Processing
```

------------------------------------------------------------------------

## Quick Troubleshooting

If an Integration Services API is not working, check in this order:

1.  **ConnRegistration** -- Verify the `IntegrationServices` connection
    points to the correct database.
2.  **SYS_PARAM** -- Verify the API username/password configuration.
3.  **Login API** -- Confirm it returns `200 OK` and an `AuthToken`.
4.  **Import URL** -- Verify `/api/Import/{BOType}` is correct.
5.  **IntegrationServicesPluginRegistry** -- Verify Warehouse, BOType,
    IP, Port, URLPath and DBName.
6.  **SCEXPERTCONNECTPLUGINS** -- Verify the corresponding plugin and
    BOType exist.
7.  **IP/Port** -- Verify the target REST API plugin is running on the
    configured IP and port.

### Quick Flow

``` text
Login Failed?
    -> Check SYS_PARAM

Login Works but Import Fails?
    -> Check AuthToken

Request Not Reaching Plugin?
    -> Check BOType
    -> Check IntegrationServicesPluginRegistry

BOType Correct?
    -> Check SCEXPERTCONNECTPLUGINS
    -> Check IP / Port / URLPath
```
