# AgentForce — Salesforce APIs Reference

- **Salesforce API version:** `v62.0` (constant `AgentForceClient.SalesforceApiVersion`)
- **Auth model:** OAuth2 `client_credentials` grant against the org's instance URL using a Connected App's consumer key/secret stored on the `AgentRegistryConnection`.
- **Authoritative base URL after auth:** `SalesforceTokenResponse.ApiInstanceUrl` (falls back to `InstanceUrl` when null). All subsequent calls use this — *not* the originally configured instance URL.
- **Pagination:** SOQL list endpoints (`/query`) return `done` + `nextRecordsUrl`. Callers loop until `done == true`.

## Endpoint Summary

| # | Method | Path | Purpose | Client method | Calling flow |
|---|--------|------|---------|---------------|--------------|
| 1 | `POST` | `/services/oauth2/token` | Acquire bearer token | `GetAccessTokenAsync` | All flows |
| 2 | `GET`  | `/services/data/v62.0/query?q=...BotDefinition` | List agents | `ListAgentsAsync` | Test connection, Sync |
| 3 | `GET`  | `/services/data/v62.0/query?q=...BotVersion`    | List bot versions (latest first) | `ListBotVersionsAsync` | Sync (status enrichment) |
| 4 | `POST` | `/services/data/v62.0/connect/bot-versions/{botVersionId}/activation` | Activate/Deactivate a version | `SetBotVersionStatusAsync` | Activate, Deactivate |
| 5 | `DELETE` | `/services/data/v62.0/sobjects/BotDefinition/{botId}` | Delete an agent | `DeleteBotAsync` | Undeploy |

---

## 1. OAuth2 Token — `GetAccessTokenAsync`

**Endpoint**

```
POST {instanceUrl}/services/oauth2/token
Content-Type: application/x-www-form-urlencoded
```

**Form body**

| Field | Value |
|-------|-------|
| `grant_type` | `client_credentials` |
| `client_id` | Connected App consumer key (from `AgentRegistryConnection.AccessKey`) |
| `client_secret` | Connected App consumer secret (from `AgentRegistryConnection.SecretKey`) |

**Response — `SalesforceTokenResponse`**

| JSON property | Type | Notes |
|---------------|------|-------|
| `access_token` | string | Bearer token to use on subsequent calls |
| `instance_url` | string | Org instance URL |
| `api_instance_url` | string | API-specific instance URL; **use this** if non-null |
| `token_type` | string | Typically `Bearer` |
| `issued_at` | string | Epoch milliseconds (as string) |
| `scope` | string | Granted scopes |

**Status code mapping**

| HTTP | UnifiedDeploymentStatusCode |
|------|-----------------------------|
| 401 / 403 / other | `AgentForceTokenAcquisitionError` |
| 429 | `ConnectedPlatformThrottled` |
| HttpRequestException / TaskCanceledException / other | `AgentForceTokenAcquisitionError` |

---

## 2. List Agents — `ListAgentsAsync`

Lists Salesforce `BotDefinition` SObject records, which represent Agentforce agents.

**Endpoint (first page)**

```
GET {apiInstanceUrl}/services/data/v62.0/query?q={urlEncodedSOQL}
Authorization: Bearer {accessToken}
```

**SOQL**

```sql
SELECT Id, DeveloperName, MasterLabel, Type, AgentType, CreatedDate, LastModifiedDate
FROM BotDefinition
```

**Endpoint (subsequent pages)**

When the previous response returned `done=false` and a `nextRecordsUrl` (a relative path like `/services/data/v62.0/query/01g...-2000`):

```
GET {apiInstanceUrl}{nextRecordsUrl}
Authorization: Bearer {accessToken}
```

**Response — `ListSalesforceAgentsResponse`**

| JSON property | Type | Notes |
|---------------|------|-------|
| `totalSize` | int | Total matching records |
| `done` | bool | `false` ⇒ another page exists |
| `records` | `SalesforceAgent[]` | Page of agents |
| `nextRecordsUrl` | string | Present only when `done == false` |

**`SalesforceAgent` record**

| JSON property | Type | Notes |
|---------------|------|-------|
| `Id` | string | Salesforce BotDefinition record ID — used as `SourceAgentId` and `botId` |
| `DeveloperName` | string | Stable API name |
| `MasterLabel` | string | Human-readable label |
| `Type` | string | e.g. `InternalCopilot` |
| `AgentType` | string | e.g. `Employee`, `AgentforceEmployeeAgent` |
| `CreatedDate` | string (ISO 8601) | Salesforce creation timestamp; parsed into target `AppDetail.CreatedDateTime` at `AgentForceService.cs:872, 913` |
| `LastModifiedDate` | string (ISO 8601) | Parsed into `LastModifiedDateTime` |

**Status code mapping**

| HTTP | UnifiedDeploymentStatusCode |
|------|-----------------------------|
| 401 / 403 | `ConnectedPlatformUnauthorizedAcess` |
| 429 | `ConnectedPlatformThrottled` |
| Other / exception | `InternalServerError` |

---

## 3. List BotVersions — `ListBotVersionsAsync`

Lists `BotVersion` SObject records across the org. A single `BotDefinition` can have multiple versions; only one may be `Active` at a time.

**Endpoint (first page)**

```
GET {apiInstanceUrl}/services/data/v62.0/query?q={urlEncodedSOQL}
Authorization: Bearer {accessToken}
```

**SOQL**

```sql
SELECT Id, DeveloperName, BotDefinitionId, Status, VersionNumber
FROM BotVersion
ORDER BY CreatedDate DESC
```

The `ORDER BY CreatedDate DESC` guarantees the **latest version per BotDefinition appears first**, which lets `AgentForceService.FetchBotVersionMapAsync` (line 937) keep only the first match per `BotDefinitionId`.

**Endpoint (subsequent pages)** — same `nextRecordsUrl` mechanism as `ListAgentsAsync`.

**Response — `ListSalesforceBotVersionsResponse`** — identical envelope shape to `ListSalesforceAgentsResponse`, with `records` typed as `SalesforceBotVersion[]`.

**`SalesforceBotVersion` record**

| JSON property | Type | Notes |
|---------------|------|-------|
| `Id` | string | BotVersion record ID — stored in `SourceIds[BotVersionId]`; required for activate/deactivate |
| `BotDefinitionId` | string | FK to parent `BotDefinition` |
| `Status` | string | `Active` or `Inactive` — stored in `SourceIds[ExternalStatus]` |
| `DeveloperName` | string | Version's developer name |
| `VersionNumber` | int? | Human-readable version (1, 2, 3, ...); stored in `SourceIds[Version]` |

**Status code mapping** — identical to `ListAgentsAsync`.

---

## 4. Set BotVersion Status — `SetBotVersionStatusAsync`

Activates or deactivates a specific BotVersion via the Salesforce Connect API.

**Endpoint**

```
POST {apiInstanceUrl}/services/data/v62.0/connect/bot-versions/{botVersionId}/activation
Authorization: Bearer {accessToken}
Content-Type: application/json
```

`{botVersionId}` is URL-escaped via `Uri.EscapeDataString`.

**Request body**

```json
{ "status": "Active" }
```

or

```json
{ "status": "Inactive" }
```

**Response** — Salesforce returns `200 OK` on success. The client treats any `IsSuccessStatusCode` as success and ignores the body.

**Status code mapping**

| HTTP | UnifiedDeploymentStatusCode |
|------|-----------------------------|
| 401 / 403 | `ConnectedPlatformUnauthorizedAcess` |
| 404 | `NotFoundError` |
| 429 | `ConnectedPlatformThrottled` |
| Other / exception | `InternalServerError` |

**Callers**

- `AgentForceService.ActivateAppAsync` → status `"Active"`
- `AgentForceService.DeactivateAppAsync` → status `"Inactive"`

Both flows pull `BotVersionId` and `ConnectionId` from `AppDetail.SourceIds` before invoking.

---

## 5. Delete BotDefinition — `DeleteBotAsync`

Deletes a Salesforce `BotDefinition` record (the Agentforce agent itself, not just a version).

**Endpoint**

```
DELETE {apiInstanceUrl}/services/data/v62.0/sobjects/BotDefinition/{botId}
Authorization: Bearer {accessToken}
```

`{botId}` is URL-escaped via `Uri.EscapeDataString`.

**Response** — Salesforce returns `204 No Content` on success. No response body is read.

**Status code mapping**

| HTTP | UnifiedDeploymentStatusCode |
|------|-----------------------------|
| 401 / 403 | `ConnectedPlatformUnauthorizedAcess` |
| 404 | `NotFoundError` |
| 429 | `ConnectedPlatformThrottled` |
| Other / exception | `InternalServerError` |

**Caller** — `AgentForceService.UndeployAppAsync` (line 539).

---

## End-to-End Flow Sequences

### Test Connection

```
GetAccessTokenAsync  →  ListAgentsAsync (first page only, validation)
```
Used by `AgentForceService.TestConnectionAsync` to verify creds + agent read scope.

### Sync (full registry refresh)

```
GetAccessTokenAsync
  loop:
    ListAgentsAsync(nextRecordsUrl)        # paginate BotDefinitions
  loop:
    ListBotVersionsAsync(nextRecordsUrl)   # paginate BotVersions (latest-first)
  merge: per agent, attach latest BotVersionId, Status, VersionNumber
```
`FetchBotVersionMapAsync` builds a `BotDefinitionId → (latestBotVersionId, status, version)` map; each agent's `SourceIds` is populated from it.

### Activate / Deactivate

```
GetAccessTokenAsync  →  SetBotVersionStatusAsync(botVersionId, "Active" | "Inactive")
```

### Undeploy (delete agent)

```
GetAccessTokenAsync  →  DeleteBotAsync(botId)
```

---

## Source ID Mapping

After sync, each `AppDetail` carries these `SourceIds` keys populated from the Salesforce responses (set at `AgentForceService.cs:911-927`):

| `SourceIds` key | Value source |
|-----------------|--------------|
| `ConnectionId` | The `AgentRegistryConnection` row |
| `AgentRegistrationProviderType` | Literal `"AgentForce"` |
| `AgentDeveloperName` | `SalesforceAgent.DeveloperName` |
| `AgentType` | `SalesforceAgent.AgentType` |
| `InstanceUrl` | Connection's configured instance URL |
| `BotVersionId` | `SalesforceBotVersion.Id` (latest version) |
| `Version` | `SalesforceBotVersion.VersionNumber` |
| `ExternalStatus` | `SalesforceBotVersion.Status` (`Active` / `Inactive`) |

`SourceAgentId` on `AppDetail` is set to the `BotDefinition.Id` (agent id, *not* version id).

---

## Common Error Behaviors

- **Token acquisition failure** is its own dedicated status code (`AgentForceTokenAcquisitionError`); all other endpoints map auth failures to `ConnectedPlatformUnauthorizedAcess`.
- **Throttling (HTTP 429)** is uniformly mapped to `ConnectedPlatformThrottled` on every endpoint.
- **TaskCanceledException** is treated as a transient infra failure and uses the same code path as `InternalServerError` (or `AgentForceTokenAcquisitionError` for the token call).
- The `api_instance_url` returned in the token response is preferred over the configured `instanceUrl` for every subsequent call — this matters when Salesforce returns a redirected API host.
