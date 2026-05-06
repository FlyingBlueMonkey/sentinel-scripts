# CloseAllSentinelIncidents.ps1

This script authenticates to Azure using a service principal, queries Microsoft Sentinel incidents in a Log Analytics workspace, and closes matching incidents automatically.

It is designed for bulk cleanup workflows (for example, closing stale `New` incidents).

## What the Script Does

- Authenticates against Azure AD using client credentials.
- Calls the Azure Management API for Microsoft Sentinel incidents.
- Filters incidents where status is `New`.
- Optionally applies an age filter based on `createdTimeUtc`.
- Closes each matching incident by setting:
  - `status`: `Closed`
  - `classification`: `Undetermined`
  - `classificationComment`: `Closed by script`
- Follows pagination using `nextLink` until all results are processed.

## Prerequisites

- PowerShell 5.1+ or PowerShell 7+.
- Network access to:
  - `https://login.microsoftonline.com`
  - `https://management.azure.com`
- An Azure AD app registration (service principal) with:
  - `tenantId`
  - `clientId`
  - `clientSecret`
- RBAC permissions in Azure for the target Sentinel workspace (or broader scope) that allow reading and updating incidents.

## Setup

1. Create or identify an Azure AD application/service principal.
2. Create a client secret and record its value.
3. Grant the service principal appropriate Azure RBAC on the target scope.
4. Collect the required values:
   - Tenant ID
   - Client ID
   - Client Secret
   - Subscription ID
   - Resource Group Name
   - Log Analytics Workspace Name

## Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `tenantId` | string | Yes | `""` | Azure AD tenant ID used to request an access token. |
| `clientId` | string | Yes | `""` | App registration (service principal) client ID. |
| `clientSecret` | string | Yes | `""` | Client secret for the app registration. |
| `subscriptionId` | string | Yes | `""` | Azure subscription containing the Sentinel workspace. |
| `resourceGroupName` | string | Yes | `""` | Resource group that contains the workspace. |
| `workspaceName` | string | Yes | `""` | Log Analytics workspace name linked to Sentinel. |
| `incidentAgeFilter` | string | No | `"30"` | Number of days for age filtering, or `all` to disable date filtering. |

## Usage

### Basic (default: incidents older than 30 days)

```powershell
powershell .\CloseAllSentinelIncidents.ps1 \
  -tenantId "<tenant-id>" \
  -clientId "<client-id>" \
  -clientSecret "<client-secret>" \
  -subscriptionId "<subscription-id>" \
  -resourceGroupName "<resource-group-name>" \
  -workspaceName "<workspace-name>"
```

### Close incidents older than 90 days

```powershell
powershell .\CloseAllSentinelIncidents.ps1 \
  -tenantId "<tenant-id>" \
  -clientId "<client-id>" \
  -clientSecret "<client-secret>" \
  -subscriptionId "<subscription-id>" \
  -resourceGroupName "<resource-group-name>" \
  -workspaceName "<workspace-name>" \
  -incidentAgeFilter 90
```

### Close all `New` incidents (no date filter)

```powershell
powershell .\CloseAllSentinelIncidents.ps1 \
  -tenantId "<tenant-id>" \
  -clientId "<client-id>" \
  -clientSecret "<client-secret>" \
  -subscriptionId "<subscription-id>" \
  -resourceGroupName "<resource-group-name>" \
  -workspaceName "<workspace-name>" \
  -incidentAgeFilter all
```

## Example Output

```text
1 incidents closed
2 incidents closed
3 incidents closed
Next link: https://management.azure.com/subscriptions/.../incidents?api-version=2024-03-01&$skipToken=...
4 incidents closed
Next link:
```

## Notes and Behavior Details

- Only incidents currently in `New` status are targeted.
- The script updates incidents with a full `PUT` request for selected properties.
- Incident titles have single quotes removed before update to avoid malformed request bodies.
- If an API rate-limit error message is detected, the script waits 60 seconds and retries.

## Troubleshooting

- Authentication failures:
  - Validate `tenantId`, `clientId`, and `clientSecret`.
  - Confirm the secret is not expired.
- Authorization failures (403):
  - Ensure RBAC permissions are assigned at the correct scope.
- No incidents closed:
  - Confirm incidents are in `New` state.
  - Check `incidentAgeFilter` is not too restrictive.
- API throttling:
  - The script retries after 60 seconds on rate-limit errors.

## Security Guidance

- Avoid hardcoding secrets in scripts checked into source control.
- Prefer secure secret storage (for example, environment variables or Azure Key Vault).
- Use least-privilege RBAC for the service principal.
