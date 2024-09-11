# Setting Up Optional Entra External ID Login (AdHoc Solution)

## Table of Contents

- [Requirements](#requirements)
- [Setting Up Microsoft Entra External ID Applications](#setting-up-microsoft-entra-external-id-applications)
  - [Required Steps](#required-steps)
  - [Entra Admin Settings Changes](#entra-admin-settings-changes)
  - [Code Modifications](#code-modifications)
  - [App Service Settings](#app-service-settings)
  - [Optional Settings](#optional-settings)

This guide outlines the process to add optional login functionality for external users using Microsoft Entra External ID.

## Requirements

**IMPORTANT:** To enable the optional External Entra ID login, ensure you meet the following prerequisites in addition to the standard sample requirements:

- **Azure Account Permissions:** You must have the appropriate [permissions to manage applications in Microsoft Entra](https://learn.microsoft.com/entra/identity/role-based-access-control/permissions-reference#cloud-application-administrator).
- **Microsoft Entra External ID Integration:** You need an active Microsoft Entra External ID tenant. Follow [this guide](https://learn.microsoft.com/en-us/entra/external-id/customers/how-to-create-external-tenant-portal) to create the tenant and obtain the Tenant ID, Primary Domain, and Username. The Tenant ID will be used to set the `AZURE_AUTH_TENANT_ID` environment variable, the Primary Domain will be used to construct the authority URL in the code, and the Username will be required for post-deployment login checks.

Before proceeding, make sure your app is running without authentication and the chat app is functional.

## Setting Up Microsoft Entra External ID Applications

To enable External Entra ID login, two Microsoft Entra applications need to be registered: one for the client UI and another for the API server.

- The **Client UI** is implemented as a [single-page application (SPA)](https://learn.microsoft.com/entra/identity-platform/scenario-spa-app-registration).
- The **API server** uses a [confidential client](https://learn.microsoft.com/entra/identity-platform/msal-client-applications) to interact with the [Microsoft Graph API](https://learn.microsoft.com/graph/use-the-api).

### Required Steps

The easiest way to set up the two applications is by using the `azd` CLI. Pre-written scripts will automate the creation and configuration for this sample. Follow these steps:

1. Run `azd env set AZURE_USE_AUTHENTICATION true` to enable the login UI and configure App Service authentication by default.
2. Ensure access control is enabled for your search index. If the index does not exist, run `prepdocs` with `AZURE_USE_AUTHENTICATION` set to `true`. If the index already exists, use the command `pwsh ./scripts/manageacl.ps1 --acl-action enable_acls` to enable access control.
3. Set the Tenant ID for authentication by running `azd env set AZURE_AUTH_TENANT_ID <YOUR-TENANT-ID>`, where `<YOUR-TENANT-ID>` is the Entra External Tenant ID you copied earlier.
4. Since the authentication tenant ID is different from your current logged-in tenant ID, run `azd auth login --tenant-id <YOUR-TENANT-ID>` to log in with your Entra External Tenant ID.
5. Deploy the app by running `azd up`. This will create the necessary client and server applications. Manual modifications will be needed later, along with additional code changes.
6. After the deployment is complete, visit the site URL and log in using your Azure Tenant (Workforce) credentials. Once logged in, navigate to **Developer Settings** and verify the additional authentication details at the bottom of the page.

### Entra Admin Settings Changes

1. Log into the Entra Admin portal, ensure you are working in the "Entra External ID" tenant, and use the right-hand menu to navigate to **User Flows**.
2. Follow this [guide](https://learn.microsoft.com/en-us/entra/external-id/customers/how-to-user-flow-sign-up-sign-in-customers) to create a user flow. Select the "Email One-Time Passcode" option in step 6. After the initial setup, you can modify the flow to include other social login options.
3. Use this [guide](https://learn.microsoft.com/en-us/entra/external-id/customers/how-to-user-flow-add-application) to add your "Client" application to the user flow. Look for the application name starting with "Azure Search OpenAI Chat Client App."

### Code Modifications

Once the configuration and settings are complete, you will need to update the code to handle Microsoft Entra External ID authentication. Open `app/backend/core/authentication.py` and make the following changes:

**Before:**
```python
self.authority = f"https://login.microsoftonline.com/{tenant_id}"
# Depending on if requestedAccessTokenVersion is 1 or 2, the issuer and audience of the token may be different
# See https://learn.microsoft.com/graph/api/resources/apiapplication
self.valid_issuers = [
    f"https://sts.windows.net/{tenant_id}/",
    f"https://login.microsoftonline.com/{tenant_id}/v2.0",
]
self.valid_audiences = [f"api://{server_app_id}", str(server_app_id)]
# See https://learn.microsoft.com/entra/identity-platform/access-tokens#validate-the-issuer for more information on token validation
self.key_url = f"{self.authority}/discovery/v2.0/keys"
```
**After:**
```python
self.authority = f"https://xxxxx.ciamlogin.com"
# Depending on if requestedAccessTokenVersion is 1 or 2, the issuer and audience of the token may be different
# See https://learn.microsoft.com/graph/api/resources/apiapplication
self.valid_issuers = [
    f"https://sts.windows.net/{tenant_id}/",
    f"https://login.microsoftonline.com/{tenant_id}/v2.0",
    f"https://{tenant_id}.ciamlogin.com/{tenant_id}/v2.0",
]
self.valid_audiences = [f"api://{server_app_id}", str(server_app_id)]
# See https://learn.microsoft.com/entra/identity-platform/access-tokens#validate-the-issuer for more information on token validation
self.key_url = f"https://login.microsoftonline.com/{tenant_id}/discovery/v2.0/keys"
```
Make sure to update the authority URL in the modified code. Replace `xxxxx` with the first part of your External Entra ID tenant's primary domain (e.g., `your-tenant-name` from `your-tenant-name.onmicrosoft.com`), which you collected earlier during the tenant setup. 

### App Service Settings
1. Log in to the Azure portal and navigate to your App Service instance. On the "Edit Identity Provider" page, update the issuer URL from `https://login.microsoftonline.com/{tenant-id}/v2.0` to `https://xxxxx.ciamlogin.com/{tenant-id}/v2.0`. The tenant ID is already correct from when you first ran the azd up command in step 5, so no changes to the tenant ID are necessary, regardless you should validate the tenant-id reflects your Entra External Tenant ID.
2. At the bottom of the "Edit Identity Provider" page under **Additional Checks**, change the **Tenant Requirement** to "Allow requests only from the issuer tenant." This option is recommended for development instances, but you may need to adjust this based on your production requirements.
3. Deploy your app by running `azd deploy` to apply all the code modifications made above.
4. **IMPORTANT:** Running `azd up` triggers various automated authentication scripts that can override your manual App Service settings changes. Ensure you manually verify the above settings after running `azd up`, and restart the App Service to apply the modified settings.
5. After the deployment is complete, visit the site URL and log in using your Azure Tenant (External) credentials, which were collected during the Entra External Tenant setup. Once logged in, go to **Developer Settings** and verify the additional authentication details at the bottom of the page.

### Optional Settings
1. (Optional) To require access control when using the app, run azd env set AZURE_ENFORCE_ACCESS_CONTROL true. Authentication is always required to search on documents with access control assigned, regardless of if unauthenticated access is enabled or not.
2. (Optional) To allow authenticated users to search on documents that have no access controls assigned, even when access control is required, run azd env set AZURE_ENABLE_GLOBAL_DOCUMENT_ACCESS true.
3. (Optional) To allow unauthenticated users to use the app, even when access control is enforced, run azd env set AZURE_ENABLE_UNAUTHENTICATED_ACCESS true. AZURE_ENABLE_GLOBAL_DOCUMENT_ACCESS should also be set to true if you want unauthenticated users to be able to search on documents with no access control.
4. Run `azd up` to deploy the optional settings.
