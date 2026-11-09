???+ warning "Upgrading from droid 0.4.0: Graph API changes"

    Microsoft removed several properties of the custom detection rules API on 2026-10-01. droid now manages the rules on Graph **v1.0** with their replacements. Most rules and configurations keep working unchanged, except:

    | Change | What to do |
    | --- | --- |
    | `custom.impactedAssets` is no longer pushed | Rules already deployed keep the entities Microsoft converted their impactedAssets into. For new rules, and to keep entities in code, move them to [`custom.xdr_entity_mappings`](#entity-mappings), or map their log source with [`entity_mappings_groups`](#entity-mappings-per-log-source). droid warns about every rule still using it |
    | `isolateDevice` without `isolationType` | The isolation type was silently dropped before, it is now an error: set `full` or `selective` |

    Behaviour changes, without anything to do:

    - Every ATT&CK tactic tag of a rule is pushed, each with the techniques belonging to it, where only the last tactic was kept before. See [MITRE ATT&CK](#mitre-attck).
    - The device groups of `blockFile` and the isolation type of `isolateDevice` are now actually pushed.
    - The first export after the upgrade may update every rule once, to the new properties.
    - New rules get `droid-<Sigma rule id>` as their id on the platform, Microsoft requiring an id starting with a letter. Rules already deployed keep theirs.
    - A rule Microsoft auto-disabled after repeated failures is left disabled with a warning, and reported by the integrity check.
    - The integrity check compares every setting export deploys (schedule, severity, title, tactics, actions, entity mappings), where it only compared the query, the description and the enabled state. Rules changed on the portal by hand, which passed before, are now reported. A rule the check cannot read is now a failure rather than a pass, and in MSSP mode the tenants a [log source variant](../log_source_variants.md) is not deployed to are skipped, as export skips them.
    - The entities of a rule are only deployed and checked when the rule or its log source maps them. A rule mapping neither is created with its device mapped, and keeps on the platform the entities it has, Microsoft's included. See [entity mappings](#entity-mappings).
    - A rule without any ATT&CK tactic tag is deployed without tactics, with a warning, and keeps the tactics already deployed on it.
    - A rule lookup Microsoft refuses, for missing permissions for instance, is an error: a removal used to report such a rule as already removed.
    - With the `default` authentication, the configured `tenant_id` is now read.

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
| days_ago     | No        | 1             | The lookback period time when using the search feature expressed in days (from 1 to 30). Not to be confused with the lookback of the deployed rules, see [schedule](#schedule) |
| query_period | Yes       | N/A           | How often the rule runs: 1h, 3h, 12h, 24h, or 0 for continuous (near real-time), which only works on quick queries. A rule overrides it with `custom.query_period`, and a group of rules with `query_period_groups` |
| alert_prefix | No        | None          | Prefix for the exported rule name  |
| tenant_id | No        | None         | Tenant ID when using "default" |
|credential_file|No|None|A YAML file with your credentials if you do not want to use the environment variables|
|auth_cert|No|None|A certificate file (PEM) when using the certificate-based authentication|
|rule_parameters.query_period_groups|No|None|The `query_period` of the rules of a log source, see [schedule](#schedule)|
|rule_parameters.entity_mappings_groups|No|None|The entity mappings of the rules of a log source, see [entity mappings per log source](#entity-mappings-per-log-source)|

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
| actions        | list       | The actions that should be performed on rule trigger, see action section for more information |
| query_period   | 0/1h/3h/12h/24h | How often this rule runs, overriding the configuration, see [schedule](#schedule) |
| xdr_entity_mappings | list  | Maps the query columns to the entities of the alerts, see the [entity mappings](#entity-mappings) section |
| impactedAssets | list       | **Removed by Microsoft**, ignored with a warning: use `xdr_entity_mappings` or [mappings per log source](#entity-mappings-per-log-source) |

### Schedule

A deployed rule runs on its own on Microsoft XDR, at the frequency set by `query_period`, in the configuration, a `query_period_groups` entry or the rule's `custom.query_period`, the rule winning over its group and the group over the configuration:

| query_period | Runs                          |
| ------------ | ----------------------------- |
| 0            | Continuously (near real-time) |
| 1h           | Every hour                    |
| 3h           | Every 3 hours                 |
| 12h          | Every 12 hours                |
| 24h          | Every day                     |

Each run searches a window of time back from now, its lookback, which Microsoft sets from the frequency. droid does not set it: Microsoft Graph accepts a lookback, but does not return it, so droid could not check the one deployed.

???+ note

    `custom.query_period` is also read by Microsoft Sentinel, as a number of hours of lookback. A rule deployed on both platforms cannot hold a value valid for both: on Microsoft XDR, any value but those above fails the export of the rule.

### Action

An identifier is required for all actions, except the device actions, which always act on `DeviceId`. droid pushes each identifier as the advanced hunting column holding it, so the query must return that column: `initiatingProcessSHA1` needs `InitiatingProcessSHA1`, `sha1` needs `SHA1`, `accountSid` needs `AccountSid`, the device actions need `DeviceId`.

A file action can take both a SHA-1 and a SHA-256 identifier as a list, for instance `identifier: [sha1, sha256]`. `stopAndQuarantineFile` also needs `DeviceId`, the device the file is stopped on, and `isolateDevice` requires an isolation type.

Removing an action from a rule removes it from the platform on the next export. The email actions (soft or hard delete, move to Inbox, Junk or Deleted Items) have no custom field: droid leaves the ones set on the portal alone, and does not report them in the integrity check.

| Action                      | Identifier                                                   | Isolation Type  | Device Group          |
| --------------------------- | ------------------------------------------------------------ | --------------- | --------------------- |
| forceUserPasswordReset      | accountSid, initiatingProcessAccountSid, requestAccountSid, onPremSid |        |                       |
| disableUser                 | accountSid, initiatingProcessAccountSid, requestAccountSid, onPremSid |        |                       |
| markUserAsCompromised       | accountObjectId, initiatingProcessAccountObjectId, servicePrincipalId, recipientObjectId | |              |
| stopAndQuarantineFile       | sha1, initiatingProcessSHA1 (deviceId is always added)       |                 |                       |
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


### MITRE ATT&CK

droid pushes the `attack.<tactic>` and `attack.t<id>` tags of a rule as the tactics of its alerts, sub-techniques grouped under their technique. A rule without any tactic tag is deployed without tactics, with a warning, and the tactics already deployed on it are left as they are: removing every ATT&CK tag from a rule does not remove its tactics from the platform. Microsoft refuses a technique under a tactic it does not belong to, so each technique goes under the tagged tactics it belongs to in MITRE ATT&CK, or under its own tactics when the rule tags none of them. The ATT&CK data comes from pySigma, which downloads it on first use, as for Elastic.

ATT&CK v18 split Defense Evasion into Stealth and Defense Impairment, while Microsoft and Sigma still use Defense Evasion: droid pushes those techniques under Defense Evasion. Should Microsoft still refuse a technique, its ATT&CK being older than droid's, droid deploys the rule without it and warns.

### Entity mappings

Entity mappings map the columns the query returns to the entities attached to the alerts, which determine how alerts are grouped into incidents and what an analyst can pivot on. They replace `impactedAssets`, removed by Microsoft.

The entities of a rule come from, in order:

1. the custom field `xdr_entity_mappings` of the rule
2. the [entity mappings of its log source](#entity-mappings-per-log-source), in the configuration
3. neither: the rule is created with its device mapped, `hosts` with `deviceIdColumn: DeviceId`, so its query must return `DeviceId`. After that, droid leaves its entities as they are on the platform and does not compare them.

Droid deploys and enforces the mappings of the first two like any other setting: export updates them, and the integrity check reports a rule whose entities differ. A rule of the third kind keeps whatever entities it has, including the ones Microsoft added when it converted the `impactedAssets` of the rules deployed before: often more than the device alone, files and processes on `SHA256` for instance.

`xdr_entity_mappings` has the shape of the Microsoft Sentinel `entity_mappings`, but the Microsoft XDR vocabulary, which is why it has its own name: a rule can carry both.

| Entity type          | Identifiers                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------------- |
| accounts             | aadUserIdColumn, dnsDomainColumn, nameColumn, ntDomainColumn, sidColumn, upnColumn, upnSuffixColumn |
| amazonResources      | amazonResourceIdColumn                                                                      |
| azureResources       | resourceIdColumn                                                                            |
| cloudApplications    | appIdColumn, nameColumn                                                                     |
| dns                  | domainNameColumn, hostIpAddressColumn, serverIpColumn                                       |
| files                | nameColumn, sha1Column, sha256Column                                                        |
| googleCloudResources | fullResourceNameColumn                                                                      |
| hosts                | deviceIdColumn, dnsDomainColumn, nameColumn, netBiosNameColumn, ntDomainColumn              |
| ips                  | addressColumn, scopeColumn                                                                  |
| mailboxes            | primaryAddressColumn                                                                        |
| mailClusters         | queryColumn, sourceColumn                                                                   |
| mailMessages         | networkMessageIdColumn, recipientColumn, senderColumn, subjectColumn                        |
| oAuthApplications    | oAuthAppIdColumn                                                                            |
| processes            | sha1Column, sha256Column                                                                    |
| registryValues       | keyColumn, valueNameColumn                                                                  |
| securityGroups       | distinguishedNameColumn, objectIdColumn, sidColumn                                          |
| urls                 | addressColumn                                                                               |

Each entry becomes one entity: map the columns identifying the same account, host or file in the same entry. A rule returning two accounts, the initiating one and the target one for instance, has two `accounts` entries.

The mappings to start from depend on the table the rule queries:

| The rule queries | Map |
| --- | --- |
| `DeviceProcessEvents`, `DeviceFileEvents` and the other `Device*` tables | `hosts`: `deviceIdColumn` DeviceId, `nameColumn` DeviceName. `accounts`: `sidColumn` InitiatingProcessAccountSid, `upnColumn` InitiatingProcessAccountUpn. `files`: `sha256Column` SHA256, `nameColumn` FileName |
| `DeviceNetworkEvents` | The above, and `ips`: `addressColumn` RemoteIP. `urls`: `addressColumn` RemoteUrl |
| `IdentityLogonEvents`, `AADSignInEventsBeta` | `accounts`: `upnColumn` AccountUpn, `sidColumn` AccountSid. `ips`: `addressColumn` IPAddress |
| `EmailEvents` | `mailboxes`: `primaryAddressColumn` RecipientEmailAddress. `mailMessages`: `networkMessageIdColumn` NetworkMessageId, `senderColumn` SenderFromAddress |

???+ warning

    The identity and email tables have no `DeviceId`: the default device mapping does not fit the rules querying them, which must set their own `xdr_entity_mappings`.

```yaml
custom:
  xdr_entity_mappings:
    - entity_type: hosts
      field_mappings:
        - identifier: deviceIdColumn
          column_name: DeviceId
        - identifier: nameColumn
          column_name: DeviceName
    - entity_type: accounts
      field_mappings:
        - identifier: sidColumn
          column_name: InitiatingProcessAccountSid
        - identifier: upnColumn
          column_name: InitiatingProcessAccountUpn
```

An unknown entity type or identifier, or an entry without field mappings, fails the export of the rule. So does a mapping Microsoft refuses, with its reason:

- a column the query does not return: `Entity mappings reference the following column(s) which are not projected by the query output`
- an entity without enough columns to identify it. An account needs `aadUserIdColumn`, `sidColumn` or `upnColumn`, or `nameColumn` with `ntDomainColumn`, `dnsDomainColumn` or `upnSuffixColumn`: a name alone is refused.

Deleting the field from a rule gives it the mappings of its log source on the next export. Without any for its log source, the mappings deployed stay as they are: change them on the portal, or map the log source.

???+ note

    The columns must be returned by the query. With Sigma rules using `| project`, or with raw rules, check that the mapped columns are part of the output.

#### Entity mappings per log source

Which entities a rule returns depends on the table it queries, rather than on the rule: every rule on process creation events returns a device, an account and a file. `entity_mappings_groups` maps them once for a log source, and every rule of that log source takes them, unless it sets its own `xdr_entity_mappings`. A group matches the rules whose `logsource` has exactly its `product`, `category` and `service`, as `query_period_groups` does.

```toml
[platforms.microsoft_xdr.rule_parameters.entity_mappings_groups.windows_process_creation]
product = "windows"
category = "process_creation"

[[platforms.microsoft_xdr.rule_parameters.entity_mappings_groups.windows_process_creation.entity_mappings]]
entity_type = "hosts"
field_mappings = [
    { identifier = "deviceIdColumn", column_name = "DeviceId" },
    { identifier = "nameColumn", column_name = "DeviceName" },
]

[[platforms.microsoft_xdr.rule_parameters.entity_mappings_groups.windows_process_creation.entity_mappings]]
entity_type = "files"
field_mappings = [
    { identifier = "sha256Column", column_name = "SHA256" },
]
```

The mappings are the same as in `xdr_entity_mappings`, and must be returned by the query of every rule of the log source: check the pipelines of the log source project them. A group without mappings, or with an invalid one, fails when droid starts.

???+ tip

    Mapping every log source puts the entities of every rule in code, reviewed and checked by the integrity check, rather than left to Microsoft.

???+ note "Migrating from impactedAssets"

    | impactedAssets                                    | xdr_entity_mappings                               |
    | ------------------------------------------------- | ------------------------------------------------- |
    | `Device` / `deviceId`                             | `hosts` / `deviceIdColumn`: `DeviceId`            |
    | `Device` / `deviceName`                           | `hosts` / `nameColumn`: `DeviceName`              |
    | `Mailbox` / `accountUpn`                          | `mailboxes` / `primaryAddressColumn`: `AccountUpn` |
    | `User` / `accountSid`                             | `accounts` / `sidColumn`: `AccountSid`            |
    | `User` / `accountUpn`                             | `accounts` / `upnColumn`: `AccountUpn`            |
    | `User` / `accountObjectId`                        | `accounts` / `aadUserIdColumn`: `AccountObjectId` |

    As `impactedAssets` did, droid maps the device of a new rule without entity mappings. Mappings of the rule or of its log source replace that default: include `hosts` in them to keep the device.

### Disabled rules

`custom.disabled` sets the status of the rule. Microsoft may also auto-disable a rule that keeps failing: droid leaves it disabled with a warning rather than re-enabling it on every export, and the integrity check reports it, until it is fixed and re-enabled on the platform.
