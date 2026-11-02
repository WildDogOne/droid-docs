- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [x] MSSP feature
    * [x] Search in multiple tenants
    * [x] Export in multiple tenants
- [x] Detection rule actions
    * [x] forceUserPasswordReset
    * [x] disableUser
    * [x] markUserAsCompromised
    * [x] stopAndQuarantineFile
    * [x] restrictAppExecution
    * [x] initiateInvestigation
    * [x] runAntivirusScan
    * [x] collectInvestigationPackage
    * [x] isolateDevice
    * [x] blockFile
    * [x] allowFile

### Limitations

Since the PySigma backend [pySigma-backend-kusto](https://github.com/AttackIQ/pySigma-backend-kusto) does not yet support correlation rules only standard sigma and raw rules are supported.

### Authentication

Two authentication mode are supported:

- `default`: Default authentication via `az login`
- `app`: Azure Registration App

When using the default authentication:

1. Head to `portal.azure.com` and authenticate using MFA
2. Use `az cli` to fetch the authentication session

When using the Azure Registation App, load the following environment variables:

- `DROID_AZURE_TENANT_ID`: Azure tenant ID
- `DROID_AZURE_CLIENT_ID`: Client ID of the registration app
- `DROID_AZURE_CLIENT_SECRET`: Azure client secret

Alternatively you can add a credential file config to the toml file:

```toml
[platforms]
[platforms.microsoft_xdr]
credential_file = "microsoft365defender-authentication.yml"
```

The file should look like this:

```yaml
client_id: 123456
client_secret: 123456
tenant_id: 123456
```

This will take precedence over the environment variables.

For certificate-based authentication, you can add the following configuration:

```toml
[platforms]
[platforms.microsoft_xdr]
auth_cert = "your_cert_file.pem"
```

Replace `your_cert_file.pem` with the path to your certificate file. This method will use the provided certificate for authentication. The certificate password is only needed if the private key. To provide a password you can use the environment variable `DROID_AZURE_CERT_PASS` or use `cert_pass: "password"` in the credential file.

???+ note

    The PEM file specified in the `auth_cert` configuration must contain both the private key and the public key, as the public key is used for fingerprinting purposes.

The keys `workspace_id` and `workspace_name` are the base workspace declaration but this values can be replaced with the environments `DROID_AZURE_WORKSPACE_ID` and `DROID_AZURE_WORKSPACE_NAME`.

???+ note

    When using the default authentication, set the `tenant_id` in the configuration

**Custom authentication via hook**:

We have also added the ability retrieve Azure Tokens from any custom app via a hook.

If the environment variable `DROID_AZURE_TOKEN_HOOK` is set with a URL, the hook attempts to acquire the token from that URL via a `GET` request. The response should contain a JSON object with the `access_token` key, which will be used as the token.

The hook function  supports dynamic URL configuration using placeholders:

- `{TENANT_ID}`: Replaced with the provided or default tenant ID.

Additional infos:

- Uses the cached token if it is still valid and the scope matches.
- Supports retries for token retrieval, with retry count and delay.
- If `DROID_AZURE_TOKEN_X_API_KEY` environment variable is present, it will be included in the request headers.

### Permissions

The required permissions for the app registration are the following:

| Microsoft Graph | Type | Description |
| --- | --- | --- |
|   CustomDetection.ReadWrite.All       | Application | Read and write all custom detection rules |
|   ThreatHunting.Read.All       | Application | Run hunting queries |


### Supported Platforms

Both Sigma and raw rules are supported under the platform name `microsoft_xdr`.

### Main config

| Parameter    | Mandatory | Default Value | Description   |
| ------------ | --------- | ------------- | -------------- |
| search_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| export_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| days_ago     | No        | 1             | The lookback period time when using the search feature expressed in days (from 1 to 30) |
| query_period | Yes       | N/A           | The period of time the query should run over. Accepted Values are: 1h, 3h, 12h, 24h or 0 for near realtime, however this only works on quick queries, Sigma queries should always be OK, but RAW rules might not work |
| alert_prefix | No        | None          | Prefix for the exported rule name  |
| tenant_id | No        | None         | Tenant ID when using "default" |
|credential_file|No|None|A YAML file with your credentials if you do not want to use the environment variables|
|auth_cert|No|None|A certificate file (PEM) when using the certificate-based authentication|

```toml
[platforms]
[platforms.microsoft_xdr]

search_auth = "app"
export_auth = "app"
query_period = "1h"
days_ago = 3
alert_prefix = "SIGMA"

[platforms.microsoft_xdr.pipelines.process_creation]
pipelines = ["pipelines/xdr_process_creation.yml", "microsoft_xdr"]
product = "windows"
category = "process_creation"
...
```

### MSSP mode

The MSSP mode (`--mssp`) exports the detection rules to every tenant listed under `export_list_mssp`. Each entry also accepts a `variants` key, which selects the telemetry sources that tenant carries when a log source is served by more than one. See [log source variants](../log_source_variants.md#per-customer-telemetry).

```toml
[platforms.microsoft_xdr.export_list_mssp.Zoidberg]

tenant_id = "122d2a69-c233-4824-a009-a431d839d799"
customer_name = "Zoidberg"
customer_filters_directory = "filters/zoidberg/"
```

### Sigma Custom Fields

| Custom Field   | Values     | Description                                                                                   |
| -------------- | ---------- | --------------------------------------------------------------------------------------------- |
| disabled       | true/false | Set to true if you want to disable the rule                                                   |
| removed        | true/false | Set to true if you want to delete the rule                                                    |
| action         | list       | The actions that should be performed on rule trigger, see action section for more information |
| entity_mappings | list      | Configure the entity mappings of the resulting alerts, see entity mappings section for more information |

### Action

An identifier is required for all actions. However, the script will automatically add the necessary fields to the query for actions with only one selection.

| Action                      | Identifier                                                   | Isolation Type  | Device Group          |
| --------------------------- | ------------------------------------------------------------ | --------------- | --------------------- |
| forceUserPasswordReset      | accountSid, initiatingProcessAccountSid                      |                 |                       |
| disableUser                 | accountSid, initiatingProcessAccountSid                      |                 |                       |
| markUserAsCompromised       | accountObjectId, initiatingProcessAccountObjectId            |                 |                       |
| stopAndQuarantineFile       | sha1, initiatingProcessSHA1                                  |                 |                       |
| restrictAppExecution        |                                                              |                 |                       |
| initiateInvestigation       |                                                              |                 |                       |
| runAntivirusScan            |                                                              |                 |                       |
| collectInvestigationPackage |                                                              |                 |                       |
| isolateDevice               |                                                              | selective, full |                       |
| blockFile                   | sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256 |                 | List of Device Groups |
| allowFile                   | sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256 |                 | List of Device Groups |

???+ danger

    Never add both "block" and "allow" files simultaneously, as this can cause significant issues.

```yaml
custom:
  disabled: false
  actions:
    - action: forceUserPasswordReset
      identifier: accountSid #accountSid, initiatingProcessAccountSid
    - action: disableUser
      identifier: initiatingProcessAccountSid #accountSid, initiatingProcessAccountSid
    - action: markUserAsCompromised
      identifier: initiatingProcessAccountObjectId #accountObjectId, initiatingProcessAccountObjectId
    - action: stopAndQuarantineFile
      identifier: initiatingProcessSHA1 # sha1, initiatingProcessSHA1
    - action: restrictAppExecution
    - action: initiateInvestigation
    - action: runAntivirusScan
    - action: collectInvestigationPackage
    - action: isolateDevice
      isolationType: selective # full, selective

    ## Allow or Block file, never both!
    - action: blockFile
      deviceGroupNames:
      identifier: initiatingProcessSHA256 # sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256
    #- action: allowFile
    #  deviceGroupNames:
    #  identifier: initiatingProcessSHA256 # sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256
```


### Entity Mappings

Entity mappings define which columns of your detection query are attached to the
resulting alert as entities (hosts, accounts, mailboxes, files, ...).

Use the following example to configure the entity mappings in a Sigma rule:

```yaml
custom:
  # ...
  entity_mappings:
    - entity_type: hosts
      field_mappings:
        - identifier: deviceIdColumn
          column_name: DeviceId
    - entity_type: accounts
      field_mappings:
        - identifier: upnColumn
          column_name: InitiatingProcessAccountUpn
    - entity_type: mailboxes
      field_mappings:
        - identifier: primaryAddressColumn
          column_name: InitiatingProcessAccountUpn
    - entity_type: files
      field_mappings:
        - identifier: sha256Column
          column_name: InitiatingProcessSHA256
```

The `identifier` must be one of the following values for the given entity type,
the `column_name` is a column of your detection query result:

| Entity Type        | Identifier (Allowed Values)                                              |
| ------------------ | -------------------------------------------------------------------------- |
| accounts           | aadUserIdColumn, dnsDomainColumn, nameColumn, ntDomainColumn, sidColumn, upnColumn, upnSuffixColumn |
| amazonResources    | amazonResourceIdColumn                                                     |
| azureResources     | resourceIdColumn                                                           |
| cloudApplications  | appIdColumn, nameColumn                                                    |
| dns                | domainNameColumn, hostIpAddressColumn, serverIpColumn                      |
| files              | nameColumn, sha1Column, sha256Column                                       |
| googleCloudResources | fullResourceNameColumn                                                   |
| hosts              | deviceIdColumn, dnsDomainColumn, nameColumn, netBiosNameColumn, ntDomainColumn |
| ips                | addressColumn, scopeColumn                                                 |
| mailboxes          | primaryAddressColumn                                                       |
| mailClusters       | queryColumn, sourceColumn                                                  |
| mailMessages       | networkMessageIdColumn, recipientColumn, senderColumn, subjectColumn       |
| oAuthApplications  | oAuthAppIdColumn                                                           |
| processes          | sha1Column, sha256Column                                                   |
| registryValues     | keyColumn, valueNameColumn                                                 |
| securityGroups     | distinguishedNameColumn, objectIdColumn, sidColumn                         |
| urls               | addressColumn                                                              |

???+ note

    Entity mappings are optional. If you do not define them, the Microsoft XDR
    service auto-derives the entity mappings from your query columns. Define
    them explicitly when the auto-derivation does not produce the entities you
    want on the alert.

???+ warning

    The former `impactedAssets` custom field (and the `impactedAssets` property
    in raw rules) is deprecated by Microsoft in favour of entity mappings and
    is no longer pushed by droid. Migrate existing rules to `entity_mappings`.
