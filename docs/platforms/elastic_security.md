- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [x] Detection rule actions
    * [x] Index
    * [x] Server log
    * [x] Email
    * [x] Webhook

### Index detection

???+ warning

    It is necessary to specify the indexes within the Sigma pipeline or it will be defaulted to `logs-*`.

#### Data streams

Most logs live in data streams, named `<type>-<dataset>-<namespace>`. Point the pipeline at the data stream, never at its backing indices: a document showing `"_index": ".ds-logs-vault-planet.express-2026.10.29-000034"` belongs to the data stream `logs-vault-planet.express`. Elasticsearch prefixes the backing indices with `.ds-`, suffixes them with a date and a generation, and rolls them over.

| `val`                         | Matches                                                          |
| ----------------------------- | ---------------------------------------------------------------- |
| `logs-vault-planet.express`   | that data stream only                                            |
| `logs-*-planet.express`       | every dataset of the `planet.express` namespace                  |
| `logs-vault-*`                | the `vault` dataset in every namespace                           |
| `*-vault-planet.express-*`    | nothing: the name ends with the namespace, there is no `-` after |

Wildcards skip the hidden `.ds-` backing indices unless the pattern itself starts with a `.`.

When several sources share one dataset, e.g. a generic `fallback` one, the index cannot tell them apart: add a condition on a field that does, with an `add_condition` transformation in the same pipeline.

Before deploying, check what a pattern resolves to:

```
GET _resolve/index/logs-*-planet.express
```

=== "Single index"

    ```yaml
    name: Elastic Windows Process Creation
    priority: 100

    transformations:

    - id: index_set_state
      type: set_state
      key: "index"
      val: "windows-sysmon.*"
      rule_conditions:
        - type: logsource
          product: windows
          category: process_creation
    ```

=== "Multiple index"

    ```yaml
    name: Elastic Windows Process Creation
    priority: 100

    transformations:

    - id: index_set_state
      type: set_state
      key: "index"
      val:
        - "windows-sysmon.*"
        - "windows-sysmon2.*"
      rule_conditions:
        - type: logsource
          product: windows
          category: process_creation
    ```

### Authentication

The same credentials are used for Kibana (export and integrity) and for Elasticsearch (search). Set `auth_method` to one of:

- `basic`: Authenticate using username and password, loaded from the environment variables:
    - `DROID_ELASTIC_USERNAME` : Elastic username
    - `DROID_ELASTIC_PASSWORD` : Elastic password
- `api_key`: Authenticate using an Elasticsearch API key, loaded from the environment variable:
    - `DROID_ELASTIC_API_KEY` : the **encoded** API key, as returned in the `encoded` field when the key is created

```toml
[platforms.elastic]
auth_method = "api_key"
```

### Permissions

Give droid a user, or an API key, of its own with the least privileges. Kibana runs a detection rule with the API key of whoever saved it last: the rules droid exports read the source indices with droid's privileges, not with the ones of an analyst.

| Privilege                                   | Kind          | Needed by                                                          |
| ------------------------------------------- | ------------- | ------------------------------------------------------------------ |
| `read` on the indices the rules read        | Elasticsearch | search, and every run of the exported rules                        |
| Security > *Rules and Exceptions*: **All**  | Kibana        | export (create, update, remove) and integrity                      |
| Management > *Actions and Connectors*: **Read** | Kibana    | export, only when [rule actions](#rule-actions) are configured     |

No cluster privilege is needed, nor any privilege on the alerts index: Kibana writes the alerts itself. As a role, in the space the rules go to:

```
PUT _security/role/droid
{
  "cluster": [],
  "indices": [
    {"names": ["logs-*"], "privileges": ["read"]}
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_securitySolutionRulesV4.all", "feature_actions.read"],
      "resources": ["space:default"]
    }
  ]
}
```

???+ note

    - List in `indices` every index pattern of your pipelines. A rule reading an index droid cannot read runs, but finds nothing.
    - The Kibana feature names carry a version, `securitySolutionRulesV4` on Kibana 9.5. `GET kbn:/api/features` lists the ones of your Kibana, or pick *Rules and Exceptions* in *Stack Management > Roles*.
    - For an API key, create it as an administrator with the role above in its `role_descriptors`. A user creating its own API keys needs the `manage_own_api_key` cluster privilege on top.

### Kibana spaces

Droid exports to the default space, or to the [space](https://www.elastic.co/docs/deploy-manage/manage-spaces) of your choice, e.g. one per team:

```toml
[platforms.elastic]
kibana_url = "https://kibana.planet-express.local"
space = "planet-express" # (1)!
```

1.  The ID of the space, as in its URLs `/s/planet-express/app/...`, not its name.

Export, integrity and removal then only see the rules of that space: a rule of the same ID in another space is another rule. Searches run on Elasticsearch, which has no spaces.

???+ warning

    - The connectors of the [rule actions](#rule-actions) must exist in that space. Connector IDs are unique across all the spaces of Kibana while a connector is only usable in its own, so give each space's connectors IDs of their own, e.g. `planet-express-alerts-index`.
    - Grant the [privileges](#permissions) in that space: `"resources": ["space:planet-express"]`.

### Supported Platforms

For Sigma rules both `esql` (ES|QL) and `eql` (EQL) are supported. ES|QL supports every correlation type EQL does and more, see [Correlation rules](#correlation-rules), and there are no known benefits to using EQL over ES|QL.

For raw rules, both esql (ES|QL) and eql (EQL) are supported. Always use the `raw_language` custom field to specify the language used in the Sigma yaml file.

For non correlation ES|QL rules, the metadata fields are automatically added to the query for better deduplication by the Elastic backend. Correlation ES|QL rules aggregate their events and have no `_id` to deduplicate on, droid suppresses their repeated alerts instead, see [Correlation rules](#correlation-rules).

### Query format

Converted ES|QL queries read like the ones written in Kibana: the source command first, then one upper-case command per line.

```
FROM logs-endpoint.events.process-* METADATA _id, _index, _version
| WHERE process.name=="powershell.exe" AND `1vendor.process.args` LIKE "*-enc*"
```

ES|QL and EQL only accept a field name without backticks when every segment of it starts with a letter (or with `_` or `@`) and is not a reserved word. Droid quotes every other field name for you, which covers:

- a segment starting with a digit, e.g. a vendor prefix like `1vendor.category`
- a segment that is a reserved word, e.g. `delivery.by` or `rule.first` in ES|QL, `rule.sequence` in EQL
- any character outside letters, digits and `_`, e.g. `process.parent-name`

???+ note

    Upgrading to a version of droid that changes the query format updates every deployed rule on the next export, and reports an integrity mismatch for each rule until it is exported again.

### MITRE ATT&CK

The `attack.*` tags of a rule fill its MITRE ATT&CK mapping in Kibana:

- a technique is listed under every tactic it belongs to, e.g. `attack.t1078` under Initial Access, Persistence, Privilege Escalation and Stealth
- a subtechnique is nested under its technique, e.g. `attack.t1078.004`
- a tactic tag without technique, e.g. `attack.initial-access` or `attack.ta0001`, keeps its own entry

The other tags are exported as Kibana tags.

### Severity and risk score

| Sigma level   | Severity | Risk score |
| ------------- | -------- | ---------- |
| critical      | critical | 99         |
| high          | high     | 73         |
| medium        | medium   | 47         |
| low           | low      | 21         |
| informational | low      | 1          |

Elastic has no informational severity, the risk score keeps those rules apart from the low ones.

### Correlation rules

[Correlation rules](https://sigmahq.io/docs/meta/correlations.html) are supported with ES|QL, and partly with EQL. Write the correlation and the rules it references in one file:

```yaml
title: Vault Sign-In Failure
name: vault_signin_failure
# ...
logsource:
    product: vault
    service: signin_attempts
detection:
    selection:
        category: credentials_failed
    condition: selection
---
title: Vault Brute Force
# ...
correlation:
    type: event_count
    rules:
        - vault_signin_failure
    group-by:
        - target_user.email
    timespan: 10m
    condition:
        gte: 5
```

which converts to:

```
FROM logs-signin-attempts-planet-express-default METADATA _id, _index, _version
| WHERE `1vault.signinattempts.category`=="credentials_failed"
| EVAL timebucket=date_trunc(10minutes, @timestamp)
| STATS event_count=count() BY timebucket, `1vault.signinattempts.target_user.email`
| WHERE event_count >= 5
```

| Correlation type   | ES\|QL | EQL |
| ------------------ | ------ | --- |
| `event_count`      | Yes    | With a `gte` condition only, as a `sequence ... with runs=N` |
| `value_count`      | Yes    | No |
| `temporal`         | Yes    | Yes, as a `sample` |
| `temporal_ordered` | No     | No |

Droid warns and skips a correlation the language cannot express. On EQL, the backend would render `value_count` as a sequence of events sharing the counted value, the opposite of distinct values, any `event_count` condition as "at least", and `temporal_ordered` as a query Elasticsearch cannot parse.

An EQL `sequence` counts the events within its `maxspan`, a sliding window, where ES|QL counts them per [time bucket](#time-buckets). Kibana deduplicates the alerts of EQL correlations itself, the [suppression of repeated alerts](#suppression-of-repeated-alerts) is only needed by ES|QL. An EQL `sample` has no time window: the [look-back](#look-back) of the rule bounds it.

Only the correlation is deployed: the rules it references are part of its query, and a rule of the file it does not reference is not deployed either.

#### Pipelines for correlations

A log source group can convert its correlation rules with other pipelines through `pipelines_correlation`, see [Correlation rule overrides](../configuration.md#correlation-rule-overrides). The whole file is then converted with these pipelines, including the rules the correlation references, so the list must still hold the field mappings and conditions of these rules. A typical use is pointing the aggregation at a narrower index:

```toml
[platforms.elastic.pipelines.vault_signin_attempts]
pipelines = ["pipelines/elastic_vault.yml"]
pipelines_correlation = [
    "pipelines/elastic_vault.yml",
    "pipelines/elastic_vault_correlation.yml", # (1)!
]
product = "vault"
service = "signin_attempts"
```

1.  A higher priority pipeline that sets the `index` state to the sign-in data stream alone.

#### Look-back

Kibana only hands a rule the events between `from` and the time it runs. A correlation looks back over its `timespan` on top of the schedule interval and the `additional_look_back`: a 1 hour correlation scheduled every 5 minutes runs with `from: now-3900s`.

#### Suppression of repeated alerts

Each run within the look-back sees the same group again, and Kibana would raise a new alert for it every time. Unless an [alert suppression](#alert-suppression) is configured for the rule, droid suppresses the alerts of a correlation on `timebucket` and its `group-by` fields, for the duration of the look-back: the runs that follow add to the first alert instead of raising new ones.

???+ warning

    - Alert suppression needs a Platinum license or higher. On a lower license, set `suppress_correlation_alerts = false`, and expect one alert per run for as long as a group stays within the look-back.
    - Kibana suppresses on three fields at most: a correlation grouped by more than two fields is deployed without suppression, with a warning, until you pick its fields with an [alert suppression](#alert-suppression).

#### Time buckets

The correlation counts events per fixed `timespan` bucket (`date_trunc`), not over a sliding window. With a 10 minutes timespan, five failures between 09:08 and 09:12 fall in two buckets, 09:00 and 09:10, and do not match a `gte: 5` condition.

### Alert suppression

A noisy rule raises one alert per matching event. [Alert suppression](https://www.elastic.co/docs/solutions/security/detect-and-alert/suppress-detection-alerts) raises one alert per group of field values instead, e.g. one per account and per hour: the events that follow are counted on that alert, in `kibana.alert.suppression.docs_count`, and run no [action](#rule-actions). It is the counterpart of `alert.suppress.fields` and `alert.suppress.period` on [Splunk](./splunk.md).

???+ warning

    Alert suppression needs a Platinum license or higher.

| Parameter      | Mandatory | Default    | Description                                                                                                  |
| -------------- | --------- | ---------- | ------------------------------------------------------------------------------------------------------------ |
| group_by       | Yes       | N/A        | 1 to 3 fields, as named in Elasticsearch once the pipelines mapped them                                      |
| duration       | No        | None       | Suppress for this period, e.g. `90s`, `30m` or `4h`. Without it, alerts are only grouped within each rule run |
| missing_fields | No        | `suppress` | `suppress` groups the events missing a `group_by` field together, `do_not_suppress` alerts on each of them   |

A rule takes the first suppression found among:

1. its custom field `elastic_suppression`, `false` turning suppression off for the rule
2. the `suppression` of its [pipeline group](../configuration.md), each [variant](../log_source_variants.md) having its own
3. the `[platforms.elastic.suppression]` of the platform

=== "Rule"

    ```yaml
    custom:
        elastic_suppression:
            group_by:
                - 1vault.signinattempts.target_user.email
            duration: 1h
    ```

=== "Pipeline group"

    ```toml
    [platforms.elastic.pipelines.vault_signin_attempts]
    pipelines = ["pipelines/elastic_vault.yml"]
    product = "vault"
    service = "signin_attempts"
    suppression = { group_by = ["1vault.signinattempts.target_user.email"], duration = "1h" }
    ```

=== "Platform"

    ```toml
    [platforms.elastic.suppression]
    group_by = ["host.name"]
    duration = "4h"
    ```

A suppression configured for a [correlation](#correlation-rules) replaces the one droid sets on its own. Its fields can only be among the ones the correlation returns, `timebucket` and its `group-by` fields, droid warns about any other. Without `duration`, the correlation is suppressed over its look-back.

### Alerts per run

A rule run raises at most `max_signals` alerts, 100 by default. Kibana drops the others and flags the run with a warning. Raise it for the platform with `max_signals`, or for one rule with the custom field `elastic_max_signals`. Kibana caps it with [`xpack.alerting.rules.run.alerts.max`](https://www.elastic.co/docs/reference/kibana/configuration-reference/alerting-settings), 1000 by default.

### Schedule

A rule runs every `interval` and looks back over that interval, plus the `additional_look_back`, plus the `timespan` of a [correlation](#look-back). A rare, expensive rule can run hourly while a critical one runs every minute, each taking the first value found among:

| Level          | Interval                     | Additional look-back                       |
| -------------- | ---------------------------- | ------------------------------------------ |
| Rule           | custom field `elastic_interval` | custom field `elastic_additional_look_back` |
| Pipeline group | `interval`                   | `additional_look_back`                     |
| Platform       | `schedule_interval` and `schedule_interval_unit` | `additional_look_back` |

Durations are written like `90s`, `5m`, `1h` or `1d`.

=== "Rule"

    ```yaml
    custom:
        elastic_interval: 1h
        elastic_additional_look_back: 5m # (1)!
    ```

    1.  Every run covers the last 65 minutes.

=== "Pipeline group"

    ```toml
    [platforms.elastic.pipelines.windows_driver_load]
    pipelines = ["pipelines/elastic_windows.yml"]
    product = "windows"
    category = "driver_load"
    interval = "15m"
    ```

### Export and integrity

Export creates the rules missing from Kibana and updates the ones that differ from what droid would deploy, leaving the others untouched: running it again over unchanged rules changes nothing. A rule whose language changed, from ES|QL to EQL or back, is deleted and created again. The custom fields `disabled`, `building_block` and `removed` disable the rule, make it a building block, or delete it; deleting a rule already gone is not an error.

Integrity checks that each rule on Kibana is the one export would deploy, field by field, and names every field that differs:

- the query, and the index of EQL rules
- the name, description, severity, risk score, MITRE ATT&CK mapping and tags
- the schedule and look-back, `interval` and `from`
- the actions, alert suppression and `max_signals`
- whether the rule is enabled, and whether it is a building block

A rule changed on Kibana since its export, or changed in the repository and not exported yet, fails the check. So does a rule marked `removed` still on Kibana, or a rule Kibana cannot show droid.

### Main config

| Parameter              | Mandatory | Default Value | Description                                                                                                 |
| ---------------------- | --------- | ------------- | ----------------------------------------------------------------------------------------------------------- |
| kibana_url             | Yes       | N/A           | The Base URL to your Kibana, since the Security API used goes via Kibana not Elasticsearch                  |
| space                  | No        | None          | ID of the Kibana space of the rules, the default space otherwise, see [Kibana spaces](#kibana-spaces)      |
| auth_method            | No        | "basic"       | `basic` or `api_key`, see [Authentication](#authentication)                                                 |
| elastic_hosts          | No        | []            | Elasticsearch hosts, only used when Searching                                                               |
| kibana_ca              | No        | False         | Certificate Chain used on the Kibana host                                                                   |
| elastic_ca             | No        | None          | Certificate Chain used on the Elastic host, can also be the same as Kibana, only used when Searching        |
| elastic_tls_verify     | No        | False         | Used to turn on and off Certificate Validation for Elastic Connections, only used when Searching            |
| schedule_interval      | No        | "1"           | Interval at which the rules run, unless a rule or its pipeline group sets its own, see [Schedule](#schedule) |
| schedule_interval_unit | No        | "h"           | Interval unit second (s), minute (m), hour (h) or day (d)                                                   |
| additional_look_back   | No        | None          | Extra time each run looks back, e.g. "1m", so events ingested late are not missed between two runs          |
| suppress_correlation_alerts | No   | true          | Suppress the repeated alerts of correlation rules, needs a Platinum license, see [Suppression of repeated alerts](#suppression-of-repeated-alerts) |
| suppression            | No        | None          | Alert suppression of every rule, see [Alert suppression](#alert-suppression)                                |
| max_signals            | No        | 100           | Maximum number of alerts a rule run raises, see [Alerts per run](#alerts-per-run)                           |
| license                | No        | "DRL"         | The license of your rule                                                                                    |
| legacy_esql            | No        | false         | This flag is only if you are on an Elastic Version that needs the square brackets around the metadata info. |
| building_block_prefix  | No        | "BB"          | Optional prefix for Building Block Rules to distinguish from Alert rules                                    |
| alert_prefix           | No        | None          | Optional prefix to your imported rules to distinguish them                                                  |
| esql_search_range_gte  | No        | "now-1h"      | How far back an ES\|QL search looks, unless the rule sets its own search look-back                          |
| eql_search_range_gte   | No        | "now-24h"     | How far back an EQL search looks, unless the rule sets its own search look-back                             |
| esql_search_range_lte  | No        | now           | Where an ES\|QL search stops, e.g. "now-5m" to leave out the events still being ingested                    |
| eql_search_range_lte   | No        | now           | Where an EQL search stops                                                                                   |


```toml
[platforms]

[platforms.elastic]
auth_method = "basic"
kibana_url = "https://kibana.test.org"
kibana_ca = "kibana-ca.pem" # Or False to ignore certificates
elastic_ca = "elastic-ca.pem"
elastic_tls_verify = true
elastic_hosts = [
    "https://elasticsearch01.test.org:9200",
    "https://elasticsearch02.test.org:9200",
    "https://elasticsearch03.test.org:9200",
]
schedule_interval = 5
schedule_interval_unit = "m"
additional_look_back = "1m" # Each run covers the last 6 minutes
license = "DRL"
alert_prefix = "SIGMA"
legacy_esql = false

# Pipeline example
[platforms.elastic.pipelines.windows_process_creation]
pipelines = ["pipelines/ecs_pipeline.yml"]
product = "windows"
category = "process_creation"
...
```

### Rule actions

Rule actions notify, or keep track, when a rule raises alerts. Each action runs a Kibana [connector](https://www.elastic.co/docs/reference/kibana/connectors-kibana): droid attaches the actions to the rules it exports, the connectors themselves are created in Kibana beforehand, since they hold the secrets (SMTP password, webhook credentials).

| Type         | Connector     | Minimum license | Default frequency |
| ------------ | ------------- | --------------- | ----------------- |
| `index`      | `.index`      | Basic           | For each alert    |
| `server_log` | `.server-log` | Basic           | Summary per run   |
| `email`      | `.email`      | Gold            | Summary per run   |
| `webhook`    | `.webhook`    | Gold            | Summary per run   |

Declare each action under `[platforms.elastic.actions.<name>]`. Every action runs for every rule, unless a rule selects its own with the [custom field](#sigma-custom-fields) `elastic_actions`.

```toml
[platforms.elastic.actions.alerts_index]
type = "index"
connector_id = "droid-alerts-index" # (1)!

[platforms.elastic.actions.dispatch_email]
type = "email"
connector_id = "planet-express-smtp"
to = ["leela@planet-express.local"]
cc = ["hermes@planet-express.local"]
throttle = "1h" # (2)!

[platforms.elastic.actions.dispatch_webhook]
type = "webhook"
connector_id = "planet-express-dispatch"
```

1.  The ID of the connector in Kibana, as shown in *Stack Management > Connectors* or returned when the connector is created.
2.  At most one summary per hour, however often the rule runs.

```yaml
custom:
    elastic_actions:
        - dispatch_email # (1)!
```

1.  This rule only sends the email. `elastic_actions: []` runs no action at all.

#### Overrides per rule

A rule can also change the parameters of its actions, e.g. to reach another team, by mapping each action to the parameters it overrides. The others come from the configuration:

```yaml
custom:
    elastic_actions:
        dispatch_email: # (1)!
            to:
                - zoidberg@planet-express.local
            subject: "Claws out: {{context.rule.name}}"
        alerts_index: # (2)!
```

1.  Sent to Zoidberg, still with Hermes in `cc` as configured.
2.  Selected without any change.

Rule files are usually open to more people than the configuration, and an action sends what the rule found. A rule therefore only overrides what its action lets it, listed in `overridable`: nothing by default. An email action can also bind the recipients rules give it to some domains:

```toml
[platforms.elastic.actions.dispatch_email]
type = "email"
connector_id = "planet-express-smtp"
to = ["leela@planet-express.local"]
cc = ["hermes@planet-express.local"]
overridable = ["to", "subject"] # (1)!
allowed_recipient_domains = ["planet-express.local"] # (2)!
```

1.  Any parameter of the action, `per_alert`, `throttle` and `connector_id` can be listed, never its `type`.
2.  Only binds the recipients a rule gives: `to = ["mom@momcorp.example"]` in a rule fails its export, while the configuration stays free to send outside. Each recipient a rule gives must be a single plain address: an entry hiding several, such as `"mom@momcorp.example, leela@planet-express.local"`, is refused.

A rule overriding anything else fails its export, naming the parameter, and nothing of it is deployed.

???+ info "Webhook per rule"

    Splunk sets the webhook URL of a rule. On Kibana, the URL belongs to the connector, along with its credentials: create a webhook connector per target, and let the rules pick theirs.

    ```toml
    [platforms.elastic.actions.dispatch_webhook]
    type = "webhook"
    connector_id = "planet-express-dispatch"
    overridable = ["connector_id"]
    ```

    ```yaml
    custom:
        elastic_actions:
            dispatch_webhook:
                connector_id: momcorp-dispatch
    ```

    Kibana refuses a connector of another type, a rule cannot turn its webhook into an email.

#### Common parameters

| Parameter    | Mandatory | Default             | Description                                                                                       |
| ------------ | --------- | ------------------- | ------------------------------------------------------------------------------------------------- |
| type         | Yes       | N/A                 | `index`, `server_log`, `email` or `webhook`                                                       |
| connector_id | Yes       | N/A                 | ID of the Kibana connector the action runs                                                        |
| per_alert    | No        | Depends on the type | `true` runs the action for each alert, `false` once per rule run with a summary of its alerts      |
| throttle     | No        | None                | Summary only: run at most once per period, e.g. `"1h"`, instead of on each rule run with alerts   |
| overridable  | No        | []                  | Parameters a rule may [override](#overrides-per-rule)                                             |
| allowed_recipient_domains | No | None              | Email only: domains the recipients a rule gives must belong to                                    |

Any other parameter is rejected, a typo fails before anything is exported.

#### Index

Keeps the alerts in an index of your own, e.g. to archive them longer than the alerts index or to feed another tool. By default each alert is indexed whole, under `alert`, with the rule, the reason and the source event:

```json
{
  "@timestamp": "2026-11-07T18:06:45.917Z",
  "alert": {
    "kibana": {"alert": {"rule": {"name": "Vault Brute Force", "...": "..."}, "reason": "...", "severity": "medium"}},
    "1vault": {"signinattempts": {"target_user": {"email": "kif@planet-express.local"}}}
  }
}
```

With `per_alert = false`, a single document per run holds `rule_id`, `rule_name`, `severity`, `risk_score`, `alerts_count` and `results_link`. Replace the document with `document`, a table whose values can use the [action variables](#action-variables):

```toml
[platforms.elastic.actions.alerts_index]
type = "index"
connector_id = "droid-alerts-index"
document = { rule = "{{context.rule.name}}", alert_id = "{{alert.id}}" }
```

Create the index and its connector before exporting. Alerts hold hundreds of fields, the `flattened` type keeps them searchable without hitting the [field limit](https://www.elastic.co/docs/manage-data/data-store/mapping/mapping-limit-settings) of the index:

```
PUT planet-express-alerts
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "alert": {"type": "flattened"}
    }
  }
}

POST kbn:/api/actions/connector/droid-alerts-index
{
  "name": "Alerts archive",
  "connector_type_id": ".index",
  "config": {"index": "planet-express-alerts", "refresh": false, "executionTimeField": "@timestamp"}
}
```

???+ warning

    A `document` given as a string is not JSON escaped by Kibana: a rule name holding a quote turns it into invalid JSON, and the action fails in the Kibana logs only. Prefer a table, whose values are escaped.

#### Server log

Writes a line in the Kibana log, handy to check that actions run. `level` is one of `trace`, `debug`, `info` (default), `warn`, `error` or `fatal`, and `message` replaces the default line.

```toml
[platforms.elastic.actions.kitchen_log]
type = "server_log"
connector_id = "droid-server-log"
level = "warn"
```

```
POST kbn:/api/actions/connector/droid-server-log
{"name": "Server log", "connector_type_id": ".server-log"}
```

#### Email

Sends the alerts by email. At least one of `to`, `cc` or `bcc` is required. `subject` and `message` (Markdown) replace the defaults, which give the rule, its severity and risk score, the number of alerts and a link to them.

```toml
[platforms.elastic.actions.dispatch_email]
type = "email"
connector_id = "planet-express-smtp"
to = ["leela@planet-express.local"]
subject = "[{{context.rule.severity}}] {{context.rule.name}}"
```

```
POST kbn:/api/actions/connector/planet-express-smtp
{
  "name": "Planet Express SMTP",
  "connector_type_id": ".email",
  "config": {"service": "other", "host": "smtp.planet-express.local", "port": 587, "secure": false, "from": "droid@planet-express.local"},
  "secrets": {"user": "droid", "password": "..."}
}
```

See the [email connector](https://www.elastic.co/docs/reference/kibana/connectors-kibana/email-action-type) for the other services (Exchange, Outlook, ...).

???+ note

    Kibana escapes the Markdown characters of the variables in the message. The HTML part of the email reads as expected, while a text-only mail client shows the escapes, e.g. `PIZZA \- Vault Brute Force`.

#### Webhook

Posts the alerts as JSON, e.g. to a SOAR. `body` replaces the default:

=== "Summary (default)"

    ```json
    {"rule_id": "…", "rule_name": "…", "severity": "…", "risk_score": 47, "results_link": "…", "alerts_count": 2, "alerts": [{…}, {…}]}
    ```

=== "per_alert = true"

    ```json
    {"rule_id": "…", "rule_name": "…", "severity": "…", "risk_score": 47, "results_link": "…", "alert": {…}}
    ```

```toml
[platforms.elastic.actions.dispatch_webhook]
type = "webhook"
connector_id = "planet-express-dispatch"
per_alert = true
```

```
POST kbn:/api/actions/connector/planet-express-dispatch
{
  "name": "Planet Express dispatch",
  "connector_type_id": ".webhook",
  "config": {"url": "https://soar.planet-express.local/hooks/droid", "method": "post", "hasAuth": true, "authType": "webhook-authentication-basic"},
  "secrets": {"user": "droid", "password": "..."}
}
```

In the body, `{{ }}` values are JSON escaped by Kibana and `{{{ }}}` are inserted raw, which is how the alerts go in as JSON objects.

#### Action variables

Templates use the [Security rule action variables](https://www.elastic.co/docs/solutions/security/detect-and-alert/common-rule-settings#rule-notifications), among them:

| Variable                         | Value                                                         |
| -------------------------------- | ------------------------------------------------------------- |
| `{{context.rule.name}}`          | Name of the rule, with its `alert_prefix`                     |
| `{{context.rule.rule_id}}`       | ID of the Sigma rule                                          |
| `{{context.rule.severity}}`      | Severity of the rule                                          |
| `{{context.rule.risk_score}}`    | Risk score of the rule                                        |
| `{{context.rule.description}}`   | Description of the rule                                       |
| `{{state.signals_count}}`        | Number of alerts of the run                                   |
| `{{context.alerts}}`             | The alerts, as JSON: the one alert of a per alert action      |
| `{{alert.id}}`                   | ID of the alert, per alert actions only                       |
| `{{context.results_link}}`       | Link to the alerts in Kibana                                  |
| `{{date}}`                       | Time of the run                                               |

???+ note

    `context.results_link` stays empty until [`server.publicBaseUrl`](https://www.elastic.co/docs/reference/kibana/configuration-reference/general-settings#server-publicbaseurl) is set in the Kibana configuration.

### Sigma Custom Fields

The Custom Fields available with the Elastic Integration are:

| Custom Field   | Values          | Description                                                                        |
| -------------- | --------------- | ---------------------------------------------------------------------------------- |
| raw_language   | esql/eql        | The Language used on raw Rules                                                     |
| disabled       | true/false      | Set to true if you want to disable the rule                                        |
| removed        | true/false      | Set to true if you want to delete the rule                                         |
| building_block | true/false      | Set to true if your rule should be a building block                                |
| index          | list of strings | Set one or more indexes to be used. Use a list (-) if you want more than one index |
| elastic_actions | list or mapping | Names of the [rule actions](#rule-actions) the rule runs, instead of all of them, or a mapping of them to the parameters the rule overrides. `[]` for none |
| elastic_interval | duration       | How often the rule runs, e.g. `1h`, see [Schedule](#schedule)                       |
| elastic_additional_look_back | duration | Extra time each run of the rule looks back, see [Schedule](#schedule)    |
| elastic_suppression | table/false | The [alert suppression](#alert-suppression) of the rule, `false` for none          |
| elastic_max_signals | integer     | Maximum number of alerts a run of the rule raises, see [Alerts per run](#alerts-per-run) |


