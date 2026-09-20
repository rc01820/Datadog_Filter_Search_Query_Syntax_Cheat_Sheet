# Datadog Filter, Search & Query Syntax Cheat Sheet

> **Purpose:** A comprehensive field reference for Datadog engineers, administrators, NOC/SRE teams, and developers who need to know **which search, filter, and query language applies where**, plus practical syntax for monitors, hosts, metrics, logs, APM, NDM/network telemetry, Synthetics, dashboards, Events, DBM, containers/Kubernetes/processes, and APIs.
>
> **Validate before you automate.** Facet names, metric names, tag keys, and some UI capabilities vary by Datadog site, product entitlement, Agent/integration version, and organization configuration. Confirm names in **Metrics Explorer**, the relevant **facet panel**, autocomplete, or a live API response before scaling a monitor, dashboard, notification rule, or script.
>
> **Last reviewed:** September 20, 2026  
> **Example convention:** Examples use generic Neomon Labs-style tags such as `env:prod`, `app:abc`, `costcenter:eoc`, `site:bos`, and `team:noc`.
>
> **Merged edition:** Consolidates the two supplied Datadog syntax cheat sheets into one de-duplicated operational reference.
>
> **Revision 2 (September 20, 2026):** Adds **Part II** (sections 25-36) covering RUM, Error Tracking, CI Visibility, security signals, SLOs, Audit Trail, DDSQL, notebooks and saved views, tag-governance APIs, and Terraform. Also adds case-sensitivity and tokenization rules (4.6), facets vs. measures and Live Tail behavior (9.17-9.19), API encoding/pagination/rate-limit guidance (17.8), and eight further gotchas (20.11).

---

## Table of Contents

1. [Mental model: the syntax families](#1-mental-model-the-syntax-families)
2. [Side-by-side master cheat sheet](#2-side-by-side-master-cheat-sheet)
3. [Common tag model](#3-common-tag-model)
4. [Boolean operators, wildcards, escaping](#4-boolean-operators-wildcards-and-escaping)
5. [Monitor List search](#5-monitor-list-search)
6. [Monitor query filters, templates, rules, and downtime](#6-monitor-query-filters-templates-rules-and-downtime)
7. [Hosts and Infrastructure](#7-hosts-and-infrastructure)
8. [Metrics](#8-metrics)
9. [Logs](#9-logs)
10. [APM / Trace Explorer](#10-apm--trace-explorer)
11. [Network Device Monitoring and Network](#11-network-device-monitoring-and-network)
12. [Synthetics](#12-synthetics)
13. [Dashboards and template variables](#13-dashboards-and-template-variables)
14. [Events](#14-events)
15. [Database Monitoring](#15-database-monitoring)
16. [Containers, Kubernetes, and Processes](#16-containers-kubernetes-and-processes)
17. [Datadog APIs](#17-datadog-apis)
18. [Time syntax reference](#18-time-syntax-reference)
19. [Cross-product examples](#19-cross-product-examples)
20. [Common mistakes and hard-won lessons](#20-common-mistakes-and-hard-won-lessons)
21. [Recommended enterprise conventions](#21-recommended-enterprise-conventions)
22. [Quick-reference recipes](#22-quick-reference-recipes)
23. [Official references](#23-official-references)
24. [One-page decision tree](#24-one-page-decision-tree)

**Part II - extended product surfaces**

25. [Real User Monitoring (RUM) and Session Replay](#25-real-user-monitoring-rum-and-session-replay)
26. [Error Tracking](#26-error-tracking)
27. [CI Visibility and Test Optimization](#27-ci-visibility-and-test-optimization)
28. [Security signals (Cloud SIEM, Workload Protection, AAP)](#28-security-signals-cloud-siem-workload-protection-and-app-and-api-protection)
29. [Service Level Objectives](#29-service-level-objectives)
30. [Audit Trail](#30-audit-trail)
31. [DDSQL - SQL over Datadog data](#31-ddsql--sql-over-datadog-data)
32. [Notebooks, saved views, and shareable URLs](#32-notebooks-saved-views-and-shareable-urls)
33. [Tag governance surfaces](#33-tag-governance-surfaces)
34. [Terraform and query-as-code](#34-terraform-and-query-as-code)
35. [Other surfaces in brief](#35-other-surfaces-in-brief)
36. [Part II decision addendum](#36-part-ii-decision-addendum)

---

# 1. Mental model: the syntax families


## 1.1 The syntax families at a glance

Almost every Datadog search box uses one of four grammars: three search grammars, plus SQL for the newer DDSQL surfaces. Knowing which one you're in solves most "why doesn't this match" problems.

| Family | Used by | Shape | Negation | AND | Attributes |
|---|---|---|---|---|---|
| **Tag scope** | Metrics, metric monitors, APM trace metrics, NDM metrics, container metrics, DBM metrics | `{key:value,key:value}` | `!key:value` | `,` or `AND` | Tags only — no `@` |
| **Event-platform search** | Logs, Spans/Traces, Events (new), RUM, Synthetics results, DBM samples, CI, Audit | `key:value @attr:value "text"` | `-key:value` or `NOT` | space or `AND` | Reserved attrs bare, custom attrs with `@` |
| **Faceted list search** | Monitor list, Host list, Synthetics test list, Dashboard list, Containers/K8s explorers, NDM device list | `facet:value free text` | `-facet:value` or `NOT` | space or `AND` | Product-specific facets |
| **SQL (DDSQL)** | DDSQL Explorer, notebook SQL cells, `dd.*` datasets, tabular query API | `SELECT ... FROM dd.hosts WHERE ...` | `NOT`, `<>` | `AND` | SQL columns; event-search strings passed as table-function arguments |

---


## 1.2 The most important rule


Datadog does **not** have one universal query language.

Within Datadog's telemetry-query surfaces, two major query families dominate:

#### Metric-based queries

Used primarily by:

-   Metrics Explorer
-   Metric dashboard widgets
-   Metric monitors
-   Metric API queries

Typical shape:

``` text
avg:system.cpu.user{env:prod AND app:abc} by {host}
```

#### Event-based queries

Used by many Explorer-style products:

-   Logs
-   APM / Trace Explorer
-   Events
-   Synthetics Results Explorer
-   several other event-oriented products

Typical shape:

``` text
env:prod AND service:web AND @http.status_code:500
```

Then there are **object-search languages** for things such as the
Monitor List and host inventory.

The same tag may therefore appear differently depending on context.

``` text
Tag:
env:prod

Metric query:
avg:system.cpu.user{env:prod}

Log/APM/Event-style query:
env:prod

Monitor-list search for a monitor TAG:
tag:env:prod

Monitor-list search for a monitor QUERY SCOPE:
scope:env:prod

Dashboard variable:
$env

Dashboard variable value:
$env.value
```

That distinction is the key to avoiding a large percentage of Datadog
filtering mistakes.

------------------------------------------------------------------------


---

# 2. Side-by-side master cheat sheet


-------------------------------------------------------------------------------------------------------------------------------------------------------------
  Area         Primary purpose                Typical syntax                                 Boolean style             Attribute syntax     Grouping
  ------------ ------------------------------ ---------------------------------------------- ------------------------- -------------------- -------------------
  Monitor List Find monitor objects           `status:Alert AND tag:team:noc`                `AND OR NOT`              Monitor              UI facets
                                                                                                                       fields/facets        

  Metric       Select telemetry evaluated by  `{env:prod AND app:abc}`                       `AND OR NOT IN NOT IN`,   Tags                 `by {host}`
  Monitor      monitor                                                                       symbolic mode                                  

  Hosts        Find infrastructure hosts      tags, host properties, facets                  Search/UI dependent       Host                 Group/filter UI
                                                                                                                       facets/properties    

  Metrics      Query timeseries               `avg:metric{filter} by {tag}`                  Metric Boolean syntax     Tags                 `by {tag}`

  Logs         Find log events                `service:web AND @http.status_code:500`        `AND OR -`                `@attribute:value`   facets/group-by

  APM          Find spans/traces              `service:web env:prod @http.status_code:500`   `AND OR -`                `@attribute:value`   facets/group-by

  NDM          Find/filter                    tags/facets; metric queries for NDM metrics    Depends on surface        facets/tags          tags/facets
               devices/interfaces/telemetry                                                                                                 

  Synthetics   Find tests/results             `type:api AND env:prod`                        `AND OR -`                facets/attributes    facets

  Dashboards   Dynamically scope widgets      `$env`, `$app`, `$host`                        Underlying data source    `$var`, `$var.value` template group-by

  Events       Find events                    `source:github AND status:error`               `AND OR -`                `@attribute:value`   facets

  DBM          Find DB queries/samples        `service:orders env:prod @db.instance:db01`    Event-style               `@db.*`              facets

  Containers   Find live containers           `java AND NOT elasticsearch` plus tags         `AND OR NOT !`            container            tag key
                                                                                                                       fields/tags          

  APIs         Programmatic search/query      Endpoint-specific                              Endpoint-specific         Endpoint-specific    Endpoint-specific
  -------------------------------------------------------------------------------------------------------------------------------------------------------------

**Do not copy syntax blindly from one row to another.**


## 2.1 Core operators across the major surfaces


| Operation | Metrics / Tag scope | Logs / APM / Events | Monitor list search | Host list |
|---|---|---|---|---|
| Match tag | `{env:prod}` | `env:prod` | `tag:"env:prod"` | `env:prod` |
| All sources | `{*}` | `*` | *(empty)* | *(empty)* |
| AND | `{env:prod,role:web}` or `{env:prod AND role:web}` | `env:prod role:web` or `env:prod AND role:web` | `status:alert type:metric` | `env:prod role:web` |
| OR | `{env:prod OR env:staging}` | `env:(prod OR staging)` | `status:(alert OR warn)` | `env:prod OR env:staging` |
| NOT | `{!env:prod}` or `{NOT env:prod}` | `-env:prod` or `NOT env:prod` | `-muted:true` / `NOT muted:true` | `-env:prod` / `NOT env:prod` |
| IN list | `{env IN (prod,staging)}` | `env:(prod OR staging)` | `status:(alert OR warn)` | — |
| NOT IN | `{env NOT IN (dev,test)}` | `-env:(dev OR test)` | — | — |
| Wildcard | `{host:web-*}` | `host:web-*` | `web*` | `web-*` |
| Grouping | `{(env:prod OR env:staging) AND !role:db}` | `(env:prod OR env:staging) -role:db` | `(status:alert OR status:warn) -muted:true` | — |
| Attribute | *n/a* | `@http.status_code:500` | *n/a* | *n/a* |
| Numeric range | *n/a (use formula/threshold)* | `@duration:[100 TO 500]` | *n/a* | *n/a* |
| Greater/less | *n/a* | `@duration:>1000` | *n/a* | *n/a* |
| Exists | *n/a* | `@user.id:*` | *n/a* | *n/a* |
| Missing | *n/a* | `-@user.id:*` | *n/a* | *n/a* |
| Exact phrase | *n/a* | `"connection refused"` | `"disk full"` | `"web 01"` |
| Group by | `by {host}` | facet group-by in UI / `group_by` in API | *n/a* | *n/a* |


## 2.2 One concept across products


"Production, service `checkout`, errors only":

| Product | Syntax |
|---|---|
| Metric query | `sum:trace.http.request.errors{env:prod,service:checkout}.as_count()` |
| Metric monitor | `sum(last_5m):sum:trace.http.request.errors{env:prod,service:checkout}.as_count() > 10` |
| Monitor list | `tag:"env:prod" service:checkout status:alert` |
| Logs | `env:prod service:checkout status:error` |
| Log monitor | `logs("env:prod service:checkout status:error").index("*").rollup("count").last("5m") > 10` |
| Trace explorer | `env:prod service:checkout status:error` |
| Trace analytics monitor | `trace-analytics("env:prod service:checkout status:error").rollup("count").last("5m") > 10` |
| Events | `env:prod service:checkout status:error` |
| Event monitor | `events("env:prod service:checkout status:error").rollup("count").last("5m") > 0` |
| Host list | `env:prod service:checkout` |
| Containers | `env:prod service:checkout` |
| Dashboard (tpl var) | `sum:trace.http.request.errors{$env,$service}.as_count()` |
| Logs API | `{"filter":{"query":"env:prod service:checkout status:error"}}` |


> **Rule:** Do not copy syntax blindly from one product row to another. Identify the data surface first.


---


# 3. Common tag model

Datadog recommends key/value tags:

``` text
<key>:<value>
```

Examples:

``` text
env:prod
service:billing
version:2.4.1
team:noc
app:abc
costcenter:eoc
site:bos
region:us-east
role:database
```

Useful enterprise tag set:

``` text
env:
service:
version:
team:
app:
costcenter:
site:
region:
role:
owner:
criticality:
managed_by:
```

Datadog's reserved/unified service tagging keys include:

``` text
env
service
version
```

Other important correlation-oriented keys include `host`, `device`,
`source`, and `team`.

### Tag inclusion

``` text
env:prod
```

### Multiple tags

The exact Boolean syntax depends on the product.

Conceptually:

``` text
env:prod AND app:abc
```

### Exclusion

Depending on the surface:

``` text
NOT app:test
```

or:

``` text
-app:test
```

or metric symbolic syntax:

``` text
!app:test
```

------------------------------------------------------------------------


---


# 4. Boolean operators, wildcards, and escaping

## 4.1 Metric filtering

Metric advanced filtering supports functional Boolean syntax:

``` text
AND
OR
NOT
IN
NOT IN
```

Example:

``` text
env:prod AND app:abc
```

``` text
env:prod AND (app:abc OR app:xyz)
```

``` text
env:prod AND app NOT IN (test,dev)
```

``` text
app IN (abc,xyz,mno)
```

Metric filtering also has symbolic forms such as:

``` text
,
!
```

**Important:** Do not mix symbolic and functional Boolean syntax in the
same metric filter.

Bad:

``` text
env:prod AND !region:us-east
```

Prefer:

``` text
env:prod AND NOT region:us-east
```

or use a consistently symbolic expression where supported.

## 4.2 Metric wildcards

Prefix:

``` text
pod_name:web-*
```

Suffix:

``` text
cluster:*-prod
```

Substring:

``` text
region:*east*
```

Metric example:

``` text
avg:system.disk.utilized{region:*east*} by {region}
```

## 4.3 Event-style Boolean syntax

Logs, APM, Events, and several Explorer products use an event-oriented
syntax.

``` text
AND
OR
-
```

Examples:

``` text
env:prod AND service:web
```

``` text
env:prod OR env:staging
```

``` text
env:prod AND -version:beta
```

Adjacent terms frequently imply `AND`:

``` text
service:web env:prod
```

## 4.4 Parentheses

Use parentheses whenever precedence could be ambiguous:

``` text
env:prod AND (service:web OR service:api)
```

``` text
(service:web OR service:api) AND -version:beta
```

------------------------------------------------------------------------


## 4.5 Escaping and special characters


Event-platform (logs/APM/events) special characters that must be escaped with `\` or wrapped in quotes:

```text
+ - = && || > >= < <= ! ( ) { } [ ] ^ " “ ” ~ * ? : \ #   and spaces
```

Two corrections worth internalizing:

- **`/` is *not* special** in log search and does not need escaping. (RUM and some
  other explorers *do* escape `/` in path values - see 25.3 - so the habit is
  harmless, but it is not required in Logs.)
- **`#` is special** and does need escaping - easy to miss in Windows paths,
  channel names, and issue references.

The published character list varies slightly between product docs. The union
above is the safe superset; when in doubt, quote the whole value.

| Want | Syntax |
|---|---|
| Value with space | `@user.name:"Jane Doe"` or `@user.name:Jane\ Doe` |
| Value with colon | `@url:"https://x.gov"` or `@url:https\:\/\/x.gov` |
| Literal wildcard char | `@path:C\:\\Windows\\*` |
| Tag with colon in value | `tag:"image:nginx:1.27"` |
| Monitor list tag | always quote: `tag:"env:prod"` |
| Metric tag values | letters, numbers, `_ - : . /`; others converted to `_` |
| Wildcards inside quotes | not expanded — `"web*"` is literal |

URL-encode every query string sent via GET (`{`, `}`, `:`, spaces, quotes, `[`, `]`).


## 4.6 Case sensitivity, tokenization, and the `?` wildcard

More "why doesn't this match" problems come from these five rules than from
anything else in this document.

**1. Boolean operators are case sensitive.** `AND`, `OR`, `NOT` must be
uppercase in event-platform search. Lowercase `and` is treated as a search term,
so `service:web and status:error` quietly searches for the literal word `and`.

**2. Attribute and tag searches are case sensitive.** `@env:Prod` and
`@env:prod` are different values. The Agent lowercases tag values on ingestion,
but API and DogStatsD submissions may preserve case, which is how an estate ends
up with `env:Prod`, `env:PROD`, and `env:prod` as three distinct dimensions.
Normalize before submission on every path.

**3. Full-text search is case *in*sensitive.** When you cannot control case at
the source, `*:term` gives case-insensitive matching, or apply a `lowercase`
filter in the Grok parser so the parsed attribute is normalized at index time.

**4. `?` matches exactly one character.** This is the escape hatch for values
containing spaces or special characters:

```text
@my_attribute:hello?world     # matches "hello world" or "hello:world"
@my_attribute:"hello:world"   # equivalent, using quotes
@my_attribute:hello\:world    # equivalent, using escaping
```

**5. `@` is reserved.** It cannot appear in a free-text term in Log Explorer
because it always introduces an attribute search. Similarly, you cannot search
for special characters in the raw log *message* at all - parse them into an
attribute with the Grok Parser first, then search the attribute.

### Free text vs. full text

| Syntax | Searches | Case |
|---|---|---|
| `timeout` | The log message (and selected error fields) | insensitive |
| `"connection refused"` | The message, as a phrase | insensitive |
| `*:timeout` | **All** attributes, including the message | insensitive |
| `@error.message:timeout` | That one attribute, exact value semantics | **sensitive** |

Full-text search is a Log Management feature and works in monitor, dashboard,
and notebook queries - but it is not accepted everywhere a query box exists
(see 9.15 for pipeline, index, and archive filters).


---


# 5. Monitor List search

**Location:** Monitors → Manage Monitors / Monitor List

This searches **monitor definitions**, not the telemetry being
evaluated.

## 5.1 Free-text search

Search monitor title/message:

``` text
database
```

Wildcard:

``` text
*postgresql*
```

Title:

``` text
title:database
```

Message:

``` text
message:database
```

Monitor ID:

``` text
1234567
```

## 5.2 Boolean search

``` text
status:Alert AND type:metric
```

``` text
status:Alert AND (tag:app:abc OR tag:app:xyz)
```

``` text
type:metric NOT tag:env:dev
```

Supported Monitor List Boolean operators include:

``` text
AND
OR
NOT
```

Parentheses are supported.

Wildcards include:

``` text
*
?
```

Regular expressions are not supported in Monitor List search.

## 5.3 Useful Monitor List attributes

Common facets include:

``` text
status:
muted:
type:
creator:
service:
tag:
env:
scope:
metric:
notification:
```

Examples:

``` text
status:Alert
```

``` text
muted:true
```

``` text
type:metric
```

``` text
creator:user@example.com
```

``` text
service:billing
```

### Search monitor tags

If the monitor itself has:

``` text
app:abc
```

search:

``` text
tag:app:abc
```

### Search monitor query scope

If the underlying monitor query contains `env:prod`:

``` text
scope:env:prod
```

### Search by metric/check

``` text
metric:system.cpu.user
```

## 5.4 Critical distinction

These are different:

``` text
tag:app:abc
```

means:

> Find monitor definitions tagged `app:abc`.

Whereas:

``` text
scope:app:abc
```

means:

> Find monitors whose query scope includes `app:abc`.

And this:

``` text
avg:system.cpu.user{app:abc}
```

means:

> Evaluate CPU telemetry carrying `app:abc`.

Three similar-looking ideas, three different jobs.

------------------------------------------------------------------------


---

# 6. Monitor query filters, templates, rules, and downtime


A monitor's internal query uses the query language associated with its
data source.

## 6.1 Metric monitor

General pattern:

``` text
<time aggregation>(<window>):<space aggregation>:<metric>{<scope>} by {<group>} <operator> <threshold>
```

Example:

``` text
avg(last_5m):avg:system.cpu.user{env:prod AND app:abc} by {host} > 90
```

Components:

``` text
avg(last_5m)
```

time evaluation

``` text
avg:
```

space aggregation

``` text
system.cpu.user
```

metric

``` text
{env:prod AND app:abc}
```

scope/filter

``` text
by {host}
```

group

``` text
> 90
```

condition

### Multi-alert by host

``` text
avg(last_5m):avg:system.cpu.user{costcenter:eoc} by {host} > 90
```

### Filter several apps

``` text
avg(last_5m):avg:system.cpu.user{
  costcenter:eoc AND app IN (abc,xyz,mno)
} by {host} > 90
```

### Exclude lab

``` text
avg(last_5m):avg:system.cpu.user{
  env:prod AND NOT app:lab
} by {host} > 90
```

## 6.2 Log monitor

Log monitors use log search syntax inside the query.

Conceptual API/query form:

``` text
logs("<search>").index("*").rollup("count").last("5m") > 0
```

Search portion:

``` text
service:web AND env:prod AND status:error
```

Attribute:

``` text
@http.status_code:[500 TO 599]
```

## 6.3 Event monitor

Event V2 alert queries use event/log-style search.

Conceptual form:

``` text
events("<search>").rollup("count").last("5m") > 0
```

Search:

``` text
source:my-source AND status:error
```

## 6.4 Service-check monitor

General API syntax:

``` text
"check.name".over("tag:value").last(N).by("group").count_by_status()
```

Example pattern:

``` text
"some.service.check".over("env:prod").last(3).by("host").count_by_status()
```

Service checks have their own available tags and grouping dimensions.
Verify the service check exists before designing around it.

## 6.5 Composite monitor

Composite monitors reference monitor IDs:

``` text
12345 && 67890
```

Typical logical operators depend on the composite expression rather than
metric/event filtering syntax.

------------------------------------------------------------------------


## 6.6 Additional monitor query patterns


| Monitor type | Query syntax |
|---|---|
| **Metric** | `avg(last_5m):avg:system.cpu.user{env:prod} by {host} > 90` |
| Metric (change) | `pct_change(avg(last_1h),last_5m):avg:system.load.1{env:prod} > 50` |
| Metric (formula) | `avg(last_5m):100 * avg:system.disk.used{*} by {host,device} / avg:system.disk.total{*} by {host,device} > 90` |
| **Anomaly** | `avg(last_4h):anomalies(avg:system.cpu.user{env:prod}, 'agile', 2) >= 1` |
| **Outlier** | `avg(last_1h):outliers(avg:system.cpu.user{env:prod} by {host}, 'DBSCAN', 3) > 0` |
| **Forecast** | `max(next_1w):forecast(avg:system.disk.in_use{env:prod} by {host,device}, 'linear', 1) >= 0.9` |
| **Service check** | `"datadog.agent.up".over("env:prod").by("host").last(2).count_by_status()` |
| **Host (heartbeat)** | `"datadog.agent.up".over("*").by("host").last(2).count_by_status()` |
| **HTTP check** | `"http.can_connect".over("instance:portal").by("host","url").last(3).count_by_status()` |
| **Log** | `logs("service:web status:error").index("*").rollup("count").by("host").last("5m") > 10` |
| Log (cardinality) | `logs("service:web").index("main").rollup("cardinality","@usr.id").last("15m") < 1` |
| Log (measure) | `logs("service:web").index("*").rollup("avg","@duration").last("5m") > 2000000000` |
| **Event** | `events("source:nagios status:error").rollup("count").by("host").last("5m") > 0` |
| **APM metric** | `avg(last_10m):avg:trace.http.request{env:prod,service:web} > 0.5` |
| APM p99 | `percentile(last_5m):p99:trace.http.request{env:prod,service:web} > 1` |
| **Trace analytics** | `trace-analytics("service:web env:prod status:error").rollup("count").last("5m") > 50` |
| **Process** | `processes('nginx').over('env:prod').by('host').rollup('count').last('5m') < 1` |
| **Composite** | `12345678 && 23456789` / `12345678 \|\| (23456789 && !34567890)` |
| **SLO (burn)** | `burn_rate("<slo_id>").over("7d").long_window("1h").short_window("5m") > 14.4` |
| **SLO (budget)** | `error_budget("<slo_id>").over("7d") > 75` |
| **RUM** | `rum("@type:error @application.id:<id>").rollup("count").last("5m") > 20` |
| **Audit** | `audit("@evt.name:\"Monitor\" @action:deleted").rollup("count").last("5m") > 0` |
| **Watchdog** | `events("priority:all sources:watchdog tags:story_type:service,env:prod").rollup("count").last("30m") > 0` |

**Rollup windows:** `last_1m`, `last_5m`, `last_10m`, `last_15m`, `last_30m`, `last_1h`, `last_2h`, `last_4h`, `last_1d`, `last_1w` (metric) — `last("5m")` style for log/event/trace.

**Aggregators:** `avg`, `sum`, `min`, `max`, `change`, `pct_change`, `percentile`.

**Comparators:** `>`, `>=`, `<`, `<=`, `==`, `!=` *(not all types support `==`/`!=`)*.


## 6.7 Notification message template syntax


```handlebars
{{#is_alert}}CRITICAL: {{host.name}} CPU at {{value}}%{{/is_alert}}
{{#is_warning}}WARN{{/is_warning}}
{{#is_recovery}}Recovered{{/is_recovery}}
{{#is_no_data}}No data from {{host.name}}{{/is_no_data}}
{{#is_alert_to_warning}}...{{/is_alert_to_warning}}
{{^is_recovery}}Not a recovery{{/is_recovery}}

{{#is_exact_match "env.name" "prod"}}@pagerduty-noc{{/is_exact_match}}
{{#is_match "host.name" "web"}}@slack-web{{/is_match}}
{{#is_renotify}}Still broken{{/is_renotify}}
{{#is_priority "P1"}}@webhook-eocns-sms{{/is_priority}}

Variables:  {{value}} {{threshold}} {{warn_threshold}} {{last_triggered_at}}
            {{host.name}} {{host.ip}} {{env.name}} {{service.name}}
            {{device.name}} {{log.message}} {{log.attributes.<path>}}
            {{span.attributes.<path>}} {{event.title}}
            {{tag_key.name}}   ← any "by {}" group key
Handles:    @email@x.com  @slack-channel  @pagerduty-svc  @webhook-name  @oncall-team
```


## 6.8 Monitor Notification Rules (v2)


Rules match on **monitor tags**, not metric/host tags.

```text
scope:  env:prod AND app:bls
scope:  team:noc AND (priority:p1 OR priority:p2)
scope:  NOT env:dev
```


> Notification Rules match **monitor tags**, not the tags on the host or metric unless those same tags are also applied to the monitor definition.


## 6.9 Downtime scopes


```text
env:prod
host:web-01 OR host:web-02
env:prod AND NOT role:db
monitor_tags: ["app:bls","env:prod"]      # target monitors by their tags
```

---


---


# 7. Hosts and Infrastructure

**Location:** Infrastructure → Hosts

The Host List is an inventory search surface.

Datadog can filter hosts using:

-   hostname/name
-   aliases
-   tags
-   cloud provider
-   environment
-   region
-   resource type
-   instance type
-   OS
-   OS version
-   Agent information
-   Docker information
-   team
-   telemetry source
-   hardware properties
-   metric-value filters

## 7.1 Tag-oriented examples

``` text
env:prod
```

``` text
app:abc
```

``` text
costcenter:eoc
```

``` text
role:database
```

A common operational workflow is:

``` text
Filter: costcenter:eoc
Group by: app
```

or:

``` text
Filter: env:prod
Group by: availability-zone
```

## 7.2 Host metric filtering

The modern Host List can also select a metric and a value range.

Conceptually:

``` text
Metric: system.cpu.user
Range: > 80
```

This is different from typing a metric timeseries query into Metrics
Explorer.

## 7.3 Host API search

The Hosts API supports a `filter` parameter and searches by host name,
alias, or tag.

Conceptual request:

``` text
GET /api/v1/hosts?filter=env:prod
```

For exact API behavior, URL encoding, authentication, pagination, and
site hostname, use the API documentation for your Datadog site.

------------------------------------------------------------------------


## 7.4 Host-down detection


```text
# Service check (preferred)
"datadog.agent.up".over("env:prod").by("host").last(2).count_by_status()
```

Or use the native **Host** monitor type. Don't rely on `datadog.agent.running` — it can hold its last value after the Agent dies.


## 7.5 Fleet Automation search examples


```text
agent_version:7.5* env:prod
os:windows
integration:snmp
-remote_config_enabled:true
```

---


---


# 8. Metrics

Metric syntax is one of the most important Datadog query languages.

## 8.1 Basic anatomy

``` text
<aggregation>:<metric>{<scope>} by {<group>}
```

Example:

``` text
avg:system.cpu.user{env:prod} by {host}
```

## 8.2 All data

``` text
avg:system.cpu.user{*}
```

## 8.3 One tag

``` text
avg:system.cpu.user{env:prod}
```

## 8.4 Multiple tags

``` text
avg:system.cpu.user{env:prod AND app:abc}
```

## 8.5 OR

``` text
avg:system.cpu.user{
  env:prod AND (app:abc OR app:xyz)
}
```

## 8.6 IN

``` text
avg:system.cpu.user{
  app IN (abc,xyz,mno)
}
```

## 8.7 NOT

``` text
avg:system.cpu.user{
  env:prod AND NOT app:test
}
```

## 8.8 NOT IN

``` text
avg:system.cpu.user{
  env:prod AND app NOT IN (test,lab,dev)
}
```

## 8.9 Group by host

``` text
avg:system.cpu.user{env:prod} by {host}
```

## 8.10 Group by multiple dimensions

Where the UI/query type supports multiple grouping dimensions:

``` text
... by {host,app}
```

## 8.11 Wildcard

``` text
avg:system.cpu.user{host:web-*} by {host}
```

Substring:

``` text
avg:system.cpu.user{region:*east*} by {region}
```

## 8.12 Excluding devices/filesystems

Example:

``` text
avg:system.disk.in_use{!device:/dev/loop*} by {host,device}
```

This is a good example of the symbolic metric filter style.

## 8.13 Aggregators

Common space aggregators include:

``` text
avg:
sum:
min:
max:
```

The appropriate aggregator depends on the metric's semantics.

Do not automatically use `sum` just because you want "all hosts." For
percentages, rates, gauges, and counts, aggregation meaning matters.

## 8.14 Query functions

Metric queries can also apply functions. Examples encountered across
Datadog include rollups, rates, smoothing, timeshifts, exclusions,
anomaly/outlier functions, and arithmetic/formulas.

Keep this distinction clear:

``` text
FILTER
```

selects timeseries.

``` text
GROUP BY
```

splits selected timeseries.

``` text
FUNCTION
```

transforms the resulting data.

------------------------------------------------------------------------


## 8.15 Component and modifier reference


| Part | Options |
|---|---|
| Aggregator (space) | `avg`, `sum`, `min`, `max` |
| Percentile (distributions) | `p50`, `p75`, `p90`, `p95`, `p99`, `p99.9` |
| Distribution aggregations | `count`, `sum`, `avg`, `min`, `max`, `pXX` |
| Scope | `{*}`, `{env:prod}`, `{env:prod,role:web}`, `{env IN (a,b)}`, `{!env:dev}`, `{host:web-*}` |
| Group | `by {host}`, `by {host,device}` |
| Rollup | `.rollup(avg)`, `.rollup(sum, 300)`, `.rollup(max, 3600)` |
| Type modifiers | `.as_count()`, `.as_rate()` |
| Fill | `.fill(null)`, `.fill(zero)`, `.fill(linear)`, `.fill(last, 300)` |


## 8.16 Common metric functions


| Category | Functions |
|---|---|
| Arithmetic | `abs()`, `log2()`, `log10()`, `cumsum()`, `integral()` |
| Interpolation | `default_zero()` |
| Timeshift | `hour_before()`, `day_before()`, `week_before()`, `month_before()`, `timeshift(q, -3600)` |
| Rate | `per_second()`, `per_minute()`, `per_hour()`, `dt()`, `diff()`, `derivative()`, `monotonic_diff()` |
| Smoothing | `ewma_3()` … `ewma_20()`, `median_3()` … `median_9()`, `autosmooth()` |
| Rank | `top(q, 10, 'mean', 'desc')`, `bottom(q, 5, 'max', 'asc')` |
| Count | `count_nonzero()`, `count_not_null()` |
| Algorithms | `anomalies()`, `outliers()`, `forecast()` |
| Exclusion | `exclude_null()`, `cutoff_min(q, 0)`, `cutoff_max(q, 100)`, `clamp_min()`, `clamp_max()` |
| Regression | `trend_line()`, `robust_trend()`, `piecewise_constant()` |


## 8.17 Advanced metric examples


```text
# Disk % used per host/device
100 * sum:system.disk.used{env:prod} by {host,device} / sum:system.disk.total{env:prod} by {host,device}

# Count of reporting hosts
count_nonzero(avg:system.cpu.idle{env:prod} by {host})

# Week-over-week
avg:system.load.1{env:prod}, week_before(avg:system.load.1{env:prod})

# Top 10 hosts by CPU
top(avg:system.cpu.user{env:prod} by {host}, 10, 'mean', 'desc')

# Custom counter as a count per minute
sum:app.orders.placed{env:prod}.as_count().rollup(sum, 60)

# Multiple IN clauses
avg:system.cpu.user{env IN (prod,staging) AND role NOT IN (db,cache)} by {host}
```


## 8.18 Metrics Summary search


```text
system.cpu.*              # name wildcard
*.errors                  # suffix
tag filter: env:prod      # metrics carrying that tag
```

---


---


# 9. Logs

Logs have one of Datadog's richest search syntaxes.

## 9.1 Free text

``` text
timeout
```

Free text searches selected message/error fields.

Phrase:

``` text
"connection refused"
```

## 9.2 Full-text search

Log Management supports:

``` text
*:timeout
```

This searches across log attributes, including message.

Prefix:

``` text
*:time*
```

Exact phrase:

``` text
*:"connection refused"
```

Full-text search has restrictions on where it can be used. It is not
interchangeable with every pipeline/index/archive filter.

## 9.3 Reserved attributes

Common reserved attributes do not require `@`:

``` text
service:web
status:error
host:web01
source:nginx
```

Example:

``` text
service:web AND status:error
```

## 9.4 Custom attributes

Use `@`:

``` text
@http.status_code:500
```

``` text
@network.client.ip:10.1.2.3
```

``` text
@user.id:12345
```

Nested:

``` text
@http.url_details.path:/api/v1/orders
```

## 9.5 Boolean

``` text
service:web AND env:prod
```

``` text
service:web OR service:api
```

Exclude:

``` text
env:prod AND -version:beta
```

## 9.6 Tag OR

``` text
env:(prod OR staging)
```

## 9.7 Wildcards

``` text
service:web*
```

``` text
service:*mongo
```

``` text
*NETWORK*
```

Wildcards inside quotes are generally treated literally rather than as
wildcards.

## 9.8 Attribute existence

Has attribute:

``` text
@http.status_code:*
```

Does not have attribute:

``` text
-@http.status_code:*
```

## 9.9 Numeric comparison

A numerical attribute generally needs to be a numerical facet for
numeric operators.

``` text
@http.response_time:>100
```

``` text
@duration:>=500
```

## 9.10 Range

``` text
@http.status_code:[400 TO 499]
```

``` text
@duration:[100 TO 500]
```

## 9.11 CIDR

Logs support `CIDR()` searches.

``` text
CIDR(@network.client.ip,10.0.0.0/8)
```

Multiple networks:

``` text
CIDR(@network.client.ip,10.0.0.0/8,192.168.0.0/16)
```

Exclude:

``` text
NOT(CIDR(@network.client.ip,10.0.0.0/8))
```

This can be extremely useful for network/security troubleshooting.

## 9.12 Special characters

Special characters may require escaping or quoting.

Example:

``` text
@my_attribute:hello\:world
```

or:

``` text
@my_attribute:"hello:world"
```

------------------------------------------------------------------------


## 9.13 Log status values


`emergency`, `alert`, `critical`, `error`, `warn`, `notice`, `info`, `debug`, `ok`


## 9.14 Operational examples


```text
service:nginx @http.status_code:[500 TO 599] -@http.url_details.path:/health
source:windows @evt.id:(4625 OR 4740) env:prod
host:eoc-* status:error "OutOfMemory"
@duration:>2000000000 service:api env:prod
CIDR(@network.client.ip,10.10.0.0/16) @http.method:POST
source:snmp_traps @snmpTrapOID:*linkDown*
```

## 9.15 Pipeline, index, exclusion, archive, and scanner filters


Same syntax:

```text
Index filter:      env:prod -source:debug
Exclusion filter:  service:healthcheck @http.url_details.path:/ping
Pipeline filter:   source:(nginx OR apache)
Archive filter:    env:prod
Sensitive Data Scanner scope: service:payments
```

## 9.16 Log-based metrics


```text
Filter:    service:web status:error
Measure:   count  | @duration (distribution)
Group by:  service, env, @http.status_code
```


---


## 9.17 Facets vs. measures

A **facet** is an indexed string dimension. A **measure** is an indexed numeric
dimension, optionally carrying a unit. The distinction decides what you are
allowed to do with an attribute:

| Operation | Needs a facet/measure? |
|---|---|
| `@attr:value` exact match | No - works on any attribute |
| `@attr:*` existence | No |
| `@attr:>100`, `@attr:[1 TO 5]` | **Yes** - must be a measure |
| Group by in the explorer | **Yes** |
| Group by in a log-based metric | **Yes** |
| Sort a column by value | **Yes** |

Units matter: a measure defined in seconds accepts `@view.loading_time:[1s TO 3s]`,
while the standard log attribute `@duration` is in **nanoseconds**
(`@duration:>1000000000` is one second). Mixing the two is a common source of
monitors that never fire.

Every facet and measure you create is also a dimension someone will later group
a monitor by, so treat the facet list as part of the tag vocabulary in Section
21, not as a per-user preference.

## 9.18 Indexes, Live Tail, and what "no results" actually means

Log search runs against **indexed** logs in the indexes you have selected. Four
different failures all look identical in the UI:

1. **Not ingested.** The Agent or integration never sent it.
2. **Ingested but excluded.** An exclusion filter dropped it at the index, so no
   query will ever find it.
3. **Indexed elsewhere.** Wrong index selected, or the time range excludes it.
4. **Query error.** Case mismatch, missing facet for a numeric operator, or an
   unescaped special character.

Diagnose in that order. **Live Tail** streams logs as they arrive, before index
routing and exclusion filters are applied - so a log visible in Live Tail may be
entirely absent from search. That makes Live Tail the right tool to confirm
*ingestion* and the wrong tool to confirm *searchability*.

Monitors evaluate indexed logs only. `.index("*")` in a log monitor query spans
all indexes; naming a single index narrows evaluation and can silently exclude
data after an index routing change.

## 9.19 Correlating logs with traces

Log-to-trace correlation depends on the tracer injecting trace identifiers into
the log record:

```text
@dd.trace_id:<id>
@dd.span_id:<id>
service:checkout env:prod status:error @dd.trace_id:*
```

That last query - error logs that *do* carry a trace id - is the fastest way to
check whether injection is actually working before blaming the APM integration.
`service`, `env`, and `version` must match between logs and traces for the
correlation UI to link them; that is the strongest practical argument for the
unified service tagging convention in Section 21.


---


# 10. APM / Trace Explorer

Trace Explorer uses event-style search.

## 10.1 Basic search

``` text
service:web-store env:prod
```

Adjacent terms imply AND.

Equivalent explicit form:

``` text
service:web-store AND env:prod
```

## 10.2 Reserved attributes

Common APM reserved fields can be queried without `@`, including fields
such as:

``` text
env:
service:
operation_name:
resource_name:
status:
trace_id:
span_id:
type:
```

Examples:

``` text
service:checkout
```

``` text
env:prod
```

``` text
status:error
```

## 10.3 Span attributes

Custom span attributes use `@`.

``` text
@git.commit.sha:12345
```

``` text
@http.status_code:500
```

## 10.4 Infrastructure/span tags

Examples:

``` text
hostname:web-server
```

``` text
availability-zone:us-east-1a
```

``` text
container_name:api-frontend
```

## 10.5 Boolean

``` text
service:web AND env:prod
```

``` text
service:web OR service:api
```

``` text
service:web AND -status:ok
```

## 10.6 Parentheses

``` text
env:prod AND (service:web OR service:api)
```

## 10.7 Grouping

Search:

``` text
service:web-store env:prod
```

Then group by a dimension such as:

``` text
http.route
```

The search decides **which spans** are included. Grouping decides **how
the selected spans are broken apart**.

------------------------------------------------------------------------


## 10.8 Detailed Trace Explorer examples


Reserved: `env`, `service`, `operation_name`, `resource_name`, `status`, `trace_id`, `ingestion_reason`, `version`.

| Want | Syntax |
|---|---|
| Service + env | `service:checkout env:prod` |
| Errors | `status:error` |
| Operation | `operation_name:http.request` |
| Resource (quote spaces) | `resource_name:"GET /api/orders"` |
| Duration with units | `@duration:>2s` / `@duration:[100ms TO 1s]` |
| HTTP status | `@http.status_code:5*` / `@http.status_code:[500 TO 599]` |
| Error type | `@error.type:java.net.SocketTimeoutException` |
| Span kind | `@span.kind:server` |
| Peer / downstream | `@peer.service:postgres` / `@db.system:postgresql` |
| Customer tag | `@customer.tier:gold` |
| Trace ID | `trace_id:1234567890123456789` |
| Service entry spans | toggle "Service Entry Spans" (UI) |
| Retained by rule | `ingestion_reason:rule` |


## 10.9 Trace metrics


Pattern: `trace.<SPAN_NAME>.<suffix>{env:...,service:...}`

| Metric | Use |
|---|---|
| `trace.<span>.hits` | Request count (`.as_count()` or `.as_rate()`) |
| `trace.<span>.errors` | Error count |
| `trace.<span>.apdex` | Apdex |
| `trace.<span>` | Latency distribution (`p50:`, `p95:`, `p99:`) |
| `trace.<span>.hits.by_http_status` | Hits by `http.status_code` / `http.status_class` |

Example error rate:

```text
100 * sum:trace.http.request.errors{env:prod,service:web}.as_count()
    / sum:trace.http.request.hits{env:prod,service:web}.as_count()
```

Per-resource:

```text
p99:trace.http.request{env:prod,service:web} by {resource_name}
```


## 10.10 Software Catalog search


```text
team:noc
tier:1
language:python
env:prod
```

---


---

# 11. Network Device Monitoring and Network


NDM requires extra care because you may be working with several
surfaces:

1.  Device inventory
2.  Interface inventory
3.  NDM metrics
4.  SNMP-derived tags
5.  Network Path / network telemetry
6.  dashboards
7.  monitors

There is no reason to force all of those into one syntax.

## 11.1 Device inventory filtering

Use the device inventory's facets/tags to narrow devices by metadata
available in your environment.

Common operational dimensions may include:

``` text
device identity
device IP
vendor
model
device type
site/location
SNMP profile
namespace
status/reachability
custom tags
```

The exact available facets depend on what Datadog has discovered and
tagged.

A useful tagging strategy:

``` text
site:bos
network_role:core
vendor:cisco
env:prod
costcenter:eoc
```

## 11.2 NDM metric queries

Once you are querying an NDM metric in Metrics Explorer, a dashboard, or
metric monitor, use **metric syntax**:

``` text
avg:<ndm.metric>{site:bos AND network_role:core} by {device}
```

The actual metric name and available tags should be selected from your
organization's Metrics Explorer rather than guessed.

## 11.3 Interface filtering

Interfaces may expose interface-oriented dimensions such as:

``` text
device
interface
interface alias/description
interface status
speed
network/device tags
```

For large NDM deployments, normalize tags at the **device level** and,
where supported, add interface-level metadata deliberately rather than
embedding business meaning into free-form interface descriptions.

## 11.4 NDM monitor rule

If the NDM signal is a metric:

``` text
<aggregation>:<ndm.metric>{site:bos} by {device}
```

uses metric syntax.

If the signal is an event-style record, use the search syntax
appropriate to that Datadog product.

**Rule:** Identify the data source first, then choose the language.

------------------------------------------------------------------------


## 11.5 Device-list facet examples


| Facet | Example |
|---|---|
| Namespace | `device_namespace:default` |
| IP | `device_ip:10.1.1.1` / `snmp_device:10.1.1.1` |
| Vendor | `device_vendor:cisco` / `vendor:cisco` |
| Profile | `snmp_profile:cisco-catalyst` |
| Hostname | `device_hostname:core-sw-01` |
| Model | `model:C9300*` |
| Status | `status:unreachable` |
| Site/custom tag | `site:dca env:prod` |
| Subnet | `autodiscovery_subnet:10.1.0.0/24` |


## 11.6 Detailed NDM metric scopes


```text
# Device reachability
avg:snmp.device.reachable{device_namespace:default} by {snmp_device}
avg:snmp.device.unreachable{site:dca} by {snmp_device}

# Interface status (tag names vary by Agent version — verify)
sum:snmp.interface.status{status:down,device_vendor:cisco} by {snmp_device,interface}

# Interface throughput (bits/sec)
8 * sum:snmp.ifHCInOctets{snmp_device:10.1.1.1} by {interface}.as_rate()

# Interface utilization (%)
avg:snmp.ifBandwidthInUsage.rate{site:dca} by {snmp_device,interface}

# Errors / discards
sum:snmp.ifInErrors{*} by {snmp_device,interface}.as_count()

# CPU / memory
avg:snmp.cpu.usage{device_vendor:cisco} by {snmp_device}
avg:snmp.memory.usage{*} by {snmp_device}

# Poll utilization (capacity signal)
100 * avg:snmp.check_duration{*} by {snmp_device} / avg:snmp.check_interval{*} by {snmp_device}
```

Common interface tags: `interface`, `interface_alias`, `interface_index`, `snmp_host`, `snmp_profile`.


## 11.7 NDM monitor examples


```text
avg(last_5m):avg:snmp.device.reachable{site:dca} by {snmp_device} < 1
"snmp.can_check".over("device_namespace:default").by("snmp_device").last(3).count_by_status()
```


## 11.8 SNMP traps


```text
source:snmp-traps @snmpTrapOID:*linkDown*
source:snmp-traps device_namespace:default @snmpTrapName:linkDown
```


## 11.9 NetFlow


```text
@source.ip:10.1.1.5 @destination.port:443
CIDR(@destination.ip,172.16.0.0/12)
@exporter.ip:10.0.0.1
```

*(Facet paths vary — confirm in the NetFlow facet panel.)*


## 11.10 Cloud Network Monitoring


Prefix with `client_` / `server_`:

```text
client_service:web server_service:postgres
client_team:noc -server_env:dev
server_availability-zone:us-east-1a
```

---


---


# 12. Synthetics

There are two related searches to distinguish:

1.  Searching/managing Synthetic **tests**
2.  Searching Synthetic **test results**

## 12.1 Test management

Test search supports free text and facets.

Example:

``` text
type:api
```

Multiple types:

``` text
type:("api" OR "api-ssl")
```

Combine facets:

``` text
type:api AND env:prod
```

## 12.2 Results Explorer

Results Explorer supports:

``` text
AND
OR
-
```

Examples:

``` text
env:prod AND service:web
```

``` text
env:prod AND -service:test
```

## 12.3 Numerical/range search

Results Explorer supports numerical range filtering through numerical
facets.

A documented style is:

``` text
Duration:[2-10]
```

Use autocomplete/facets in the Results Explorer to obtain the exact
field name exposed by your Synthetic result data.

## 12.4 Tag strategy

Useful Synthetic tags:

``` text
env:prod
service:portal
app:abc
site:external
team:noc
criticality:1
journey:login
```

Then test management becomes much easier:

``` text
env:prod AND app:abc
```

------------------------------------------------------------------------


## 12.5 Test-list facet reference


| Facet | Example |
|---|---|
| Type | `type:api`, `type:browser`, `type:mobile` |
| Subtype | `subtype:http`, `subtype:ssl`, `subtype:dns`, `subtype:tcp`, `subtype:icmp`, `subtype:multi`, `subtype:grpc`, `subtype:websocket`, `subtype:udp` |
| State | `state:paused` / `state:live` |
| Status | `status:alert` / `status:ok` / `status:"no data"` |
| Location | `locations:aws:us-east-1` / `locations:pl:eoc-private-*` |
| Tag | `env:prod`, `app:bls` |
| Team / creator | `team:noc`, `creator:"shayne@..."` |
| Free text | `portal login` |

```text
type:api subtype:ssl env:prod status:alert
-state:paused locations:pl:*
```


## 12.6 Result Explorer attribute examples


```text
@test.type:api @result.status:failed
@test.id:abc-def-ghi
@location.id:aws:us-east-1
@result.duration:>5000
```

*(Facet names vary — check the facet panel.)*


## 12.7 Synthetics metrics


```text
sum:synthetics.test_runs{test_type:api,status:failed} by {synthetics.test_id}.as_count()
avg:synthetics.http.response.time{env:prod} by {location}
```


## 12.8 Assertion syntax


```text
statusCode       is        200
responseTime     lessThan  2000
header content-type contains application/json
body             validatesJSONPath  $.status  is  "ok"
certificate      expiresInMoreThan  30
```


## 12.9 Local and global variables


```text
{{ MY_GLOBAL_VAR }}
{{ numeric(6) }}  {{ alphabetic(8) }}  {{ uuid }}
{{ date(0d, YYYY-MM-DD) }}  {{ timestamp(0, ms) }}
```

---


---


# 13. Dashboards and template variables

Dashboard template variables are the bridge between reusable dashboards
and Datadog's underlying query languages.

## 13.1 Variable definition

Suppose the tag is:

``` text
app:abc
```

Create a template variable based on tag key:

``` text
app
```

The variable is referenced as:

``` text
$app
```

## 13.2 Variable representations

If:

``` text
$app = app:abc
```

then:

``` text
$app
```

represents key/value context.

``` text
$app.key
```

returns the key:

``` text
app
```

``` text
$app.value
```

returns:

``` text
abc
```

## 13.3 Metric widget

Conceptually:

``` text
avg:system.cpu.user{$env AND $app} by {host}
```

The dashboard UI often inserts template variables through the `from`
selector rather than requiring hand-written raw syntax.

## 13.4 Text widgets

Template variables can be displayed in text-oriented widgets.

``` text
$app
$app.key
$app.value
```

This is useful for standardized application dashboards.

Example title:

``` text
Application: $app.value
```

## 13.5 Dynamic concatenation

A value can be embedded in another expression/string where supported.

Example from Datadog's documented template-variable pattern:

``` text
env:staging-$service.value
```

## 13.6 Event overlay

Dashboard event overlay:

``` text
region:$region.value
```

Multiple template variables:

``` text
role:$role.value,env:$env.value
```

## 13.7 Filter vs Group By variables

A dashboard variable can be configured as a:

``` text
Filter
```

or:

``` text
Group By
```

Use a filter when the viewer should choose which data is in scope.

Use group-by when the viewer should dynamically choose how the data is
segmented.

## 13.8 URL representation

Template selections are reflected in dashboard URLs using parameters
similar to:

``` text
tpl_var_env=prod
```

This is useful for linking users to a pre-filtered dashboard.

------------------------------------------------------------------------


## 13.9 Dashboard list search


```text
free text:    NOC overview
filters:      Created by me, Shared, Favorites, Preset lists (UI)
```


## 13.10 Widget query examples


```text
Timeseries:  avg:system.cpu.user{$env} by {host}
Query value: count_nonzero(avg:system.cpu.idle{$env} by {host})   # formula field
Top list:    top(avg:system.disk.in_use{$env} by {host}, 10, 'max', 'desc')
Log stream:  service:web status:error $env
Monitor summary query:  tag:"env:prod" status:(alert OR warn)
SLO list:    team:noc
Event stream: source:nagios $env
Change:      avg:system.load.1{$env}   compare to: day_before
```

---


# 14. Events

Events Explorer uses log-style search syntax.

## 14.1 Source

``` text
source:github
```

Multiple:

``` text
source:(github OR chef)
```

## 14.2 Host

``` text
host:server01
```

## 14.3 Service

``` text
service:kafka
```

## 14.4 Status

``` text
status:error
```

Common documented statuses include:

``` text
error
warning
info
ok
```

## 14.5 Tag/facet

``` text
availability-zone:us-east-1a
```

## 14.6 Wildcard

``` text
container_id:foo*
```

## 14.7 Event attribute

``` text
@evt.name:foo
```

## 14.8 Boolean

``` text
source:github AND status:error
```

``` text
source:(github OR chef) AND env:prod
```

Exclude:

``` text
env:prod AND -service:test
```

------------------------------------------------------------------------


## 14.9 Legacy v1 Event Stream syntax


```text
sources:nagios,chef tags:env:prod,role:web priority:low status:error hosts:web-01
```


## 14.10 Event monitor


```text
events("source:nagios status:error env:prod").rollup("count").by("host").last("15m") > 0
```


## 14.11 Event overlays on graphs


```text
source:github env:prod
tags:deploy service:$service.value
```

---


---


# 15. Database Monitoring

DBM surfaces include Query Metrics, Query Samples, plans/explain-related
data, and database telemetry.

Filtering depends on the DBM page, but query samples use Datadog
event-style search concepts.

## 15.1 Core dimensions

Useful DBM dimensions include:

``` text
service:
host:
env:
source:
@db.instance:
@db.query_signature:
```

## 15.2 Database product

Query Samples can filter database product through `source`.

Conceptually:

``` text
source:postgres
```

or the value shown in your DBM facet list.

## 15.3 Service

``` text
service:orders-db
```

## 15.4 Environment

``` text
env:prod
```

## 15.5 Host

``` text
host:db01
```

## 15.6 Database instance

``` text
@db.instance:orders
```

## 15.7 Query signature

``` text
@db.query_signature:<signature>
```

## 15.8 Combined DBM search

``` text
service:orders-db env:prod @db.instance:orders
```

## 15.9 API-oriented DBM query-sample example

Datadog documents query samples as database-query records that can be
narrowed with fields such as:

``` text
dbm_type:activity
@db.query_signature:<signature>
service:<name>
env:<env>
host:<host>
@db.instance:<instance>
```

Example:

``` text
dbm_type:activity service:orders-db env:prod @db.instance:orders
```

------------------------------------------------------------------------


## 15.10 Detailed DBM UI filters


| Facet | Example |
|---|---|
| Engine | `dbms:postgres`, `dbms:mysql`, `dbms:sqlserver`, `dbms:oracle` |
| Host | `host:rds-prod-01` |
| Database | `db:orders` / `@db.instance:orders` |
| User | `@db.user:app_rw` |
| Query signature | `@db.query_signature:a1b2c3d4` |
| Statement text | `@db.statement:*SELECT*orders*` |
| Duration | `@duration:>1s` |
| State | `@db.state:active` / `@db.wait_event_type:Lock` |
| Tag | `env:prod` |

```text
dbms:postgres env:prod @duration:>2s @db.user:app_rw
dbms:sqlserver @db.wait_type:LCK* host:sql-prod-*
```


## 15.11 DBM metrics


```text
# Postgres
sum:postgresql.queries.count{env:prod} by {query_signature}.as_rate()
sum:postgresql.queries.time{db:orders} by {query_signature}
avg:postgresql.connections{env:prod} by {host}
max:postgresql.replication_delay{*} by {host}

# MySQL
sum:mysql.queries.count{env:prod} by {query_signature}.as_rate()
avg:mysql.performance.threads_connected{*} by {host}

# SQL Server
sum:sqlserver.queries.count{env:prod} by {query_signature}.as_rate()
avg:sqlserver.stats.connections{*} by {host}
```

Common DBM tags: `db`, `user`, `dbms`, `query_signature`, `host`.


## 15.12 DBM monitor example


```text
avg(last_10m):avg:postgresql.replication_delay{env:prod} by {host} > 30
```

Or use the **Database Monitoring** monitor type for query-level conditions.

---


---

# 16. Containers, Kubernetes, and Processes


Live Containers / Containers Explorer has both string search and tag
filtering concepts.

## 16.1 String search

String search can match container name, ID, or image fields.

``` text
java
```

## 16.2 AND

``` text
java AND elasticsearch
```

## 16.3 OR

``` text
java OR python
```

## 16.4 NOT

``` text
java NOT elasticsearch
```

or:

``` text
java !elasticsearch
```

## 16.5 Parentheses

``` text
(NOT (elasticsearch OR kafka) java) OR python
```

## 16.6 Tag filtering

Operational examples:

``` text
env:prod
```

``` text
service:api
```

``` text
kube_namespace:production
```

``` text
cluster_name:prod-cluster
```

Available tag keys depend on container runtime, orchestrator,
integrations, and tagging configuration.

## 16.7 Grouping

A powerful workflow:

``` text
Filter by: env:prod
Group by: service
```

or:

``` text
Filter by: kube_namespace:production
Group by: kube_deployment
```

------------------------------------------------------------------------


## 16.8 Containers / Kubernetes Explorer facet reference


| Facet | Example |
|---|---|
| Cluster | `kube_cluster_name:prod-eks` |
| Namespace | `kube_namespace:payments` |
| Deployment | `kube_deployment:checkout` |
| StatefulSet / DaemonSet | `kube_stateful_set:redis`, `kube_daemon_set:datadog-agent` |
| Pod | `pod_name:checkout-*` |
| Container | `container_name:nginx` |
| Image | `image_name:nginx`, `short_image:nginx`, `image_tag:1.27` |
| Docker image | `docker_image:nginx:1.27` |
| Service | `kube_service:checkout` |
| Node | `kube_node:ip-10-0-1-5*` |
| Pod phase | `pod_phase:running` / `pod_phase:pending` |
| Container ID | `container_id:abc123*` |

```text
kube_cluster_name:prod-eks kube_namespace:payments -pod_phase:running
image_name:nginx -image_tag:1.27
```


## 16.9 Container and Kubernetes metrics


```text
avg:container.cpu.usage{kube_namespace:payments} by {kube_deployment}
avg:container.memory.usage{kube_cluster_name:prod-eks} by {pod_name}
sum:kubernetes.containers.restarts{kube_namespace:payments} by {pod_name}
sum:kubernetes_state.pod.status_phase{pod_phase:pending} by {kube_namespace}
sum:kubernetes_state.deployment.replicas_available{*} by {kube_deployment}
max:kubernetes_state.container.status_report.count.waiting{reason:crashloopbackoff} by {pod_name}
avg:docker.cpu.usage{docker_image:nginx*} by {container_name}
```


## 16.10 Live Processes


```text
Search:  nginx
Tags:    env:prod host:web-*
Command: command:"java -jar app.jar"
User:    user:svc_app
```

Process monitor:

```text
processes('nginx').over('env:prod').by('host').rollup('count').last('5m') < 1
```


## 16.11 Autodiscovery annotation template variables


```yaml
ad.datadoghq.com/nginx.checks: |
  {"nginx":{"instances":[{"nginx_status_url":"http://%%host%%:%%port%%/status"}]}}
ad.datadoghq.com/nginx.logs: '[{"source":"nginx","service":"web"}]'
```

`%%host%%`, `%%port%%`, `%%env_VAR%%`, `%%pid%%`, `%%hostname%%`, `%%kube_namespace%%`.

---


---

# 17. Datadog APIs


There is **no universal Datadog API search parameter**. Each endpoint defines its own filter/query parameters and may embed one of Datadog's UI query languages.

Typical authentication headers:

```text
DD-API-KEY
DD-APPLICATION-KEY
Content-Type: application/json
```

Never hard-code production keys into repositories. Use environment variables or a secret manager.


## 17.1 Sites and base URLs


| Site | API base |
|---|---|
| US1 | `https://api.datadoghq.com` |
| US3 | `https://api.us3.datadoghq.com` |
| US5 | `https://api.us5.datadoghq.com` |
| EU1 | `https://api.datadoghq.eu` |
| AP1 | `https://api.ap1.datadoghq.com` |
| US1-FED (GovCloud) | `https://api.ddog-gov.com` |

Headers: `DD-API-KEY`, `DD-APPLICATION-KEY`, `Content-Type: application/json`.


## 17.2 Endpoint and filter syntax side-by-side


| Product | Endpoint | Filter parameter / body |
|---|---|---|
| **Monitors (search)** | `GET /api/v1/monitor/search` | `?query=status:alert tag:"env:prod"&page=0&per_page=100` |
| Monitors (list) | `GET /api/v1/monitor` | `?monitor_tags=env:prod,app:bls&tags=host:web-01&name=CPU` |
| Monitor groups | `GET /api/v1/monitor/groups/search` | `?query=status:alert` |
| Notification rules | `GET/POST /api/v2/monitor/notification_rule` | `"filter":{"tags":["env:prod"]}` |
| Downtimes | `GET/POST /api/v2/downtime` | `"scope":"env:prod AND role:web"` |
| **Hosts** | `GET /api/v1/hosts` | `?filter=env:prod&sort_field=cpu&sort_dir=desc&count=1000&start=0` |
| Host totals | `GET /api/v1/hosts/totals` | — |
| Host tags | `GET /api/v1/tags/hosts/{host}` | — |
| **Metrics (query)** | `GET /api/v1/query` | `?from=<epoch_s>&to=<epoch_s>&query=avg:system.cpu.user{env:prod} by {host}` |
| Metrics (v2 timeseries) | `POST /api/v2/query/timeseries` | `data.attributes.queries[].query` + `formulas[]` |
| Metrics (v2 scalar) | `POST /api/v2/query/scalar` | same, returns single values |
| Metric list | `GET /api/v2/metrics` | `?filter[tags]=env:prod&filter[configured]=true&window[seconds]=3600` |
| Active metrics | `GET /api/v1/metrics` | `?from=<epoch_s>&host=web-01&tag_filter=env:prod` |
| Metric tags | `GET /api/v2/metrics/{metric}/all-tags` | — |
| Submit | `POST /api/v2/series` | `series[].tags: ["env:prod"]` |
| **Logs (search)** | `POST /api/v2/logs/events/search` | `{"filter":{"query":"...","from":"now-15m","to":"now","indexes":["*"]},"sort":"-timestamp","page":{"limit":100}}` |
| Logs (GET) | `GET /api/v2/logs/events` | `?filter[query]=...&filter[from]=now-15m&page[limit]=100&page[cursor]=...` |
| Logs aggregate | `POST /api/v2/logs/analytics/aggregate` | `compute[]`, `group_by[]`, `filter.query` |
| Log submit | `POST https://http-intake.logs.<site>/api/v2/logs` | `ddsource`, `ddtags`, `service`, `hostname` |
| **Spans (search)** | `POST /api/v2/spans/events/search` | `{"data":{"type":"search_request","attributes":{"filter":{"query":"service:web status:error","from":"now-15m","to":"now"}}}}` |
| Spans (GET) | `GET /api/v2/spans/events` | `?filter[query]=...&filter[from]=now-15m` |
| Spans aggregate | `POST /api/v2/spans/analytics/aggregate` | `compute`, `group_by`, `filter` |
| **NDM devices** | `GET /api/v2/ndm/devices` | `?filter[tag]=device_namespace:default&page[size]=100&page[number]=0&sort=name` |
| NDM device | `GET /api/v2/ndm/devices/{device_id}` | `device_id` = `namespace:ip` |
| NDM interfaces | `GET /api/v2/ndm/interfaces` | `?device_id=default:10.1.1.1` |
| NDM device tags | `GET/PATCH /api/v2/ndm/tags/devices/{device_id}` | `tags[]` |
| **Synthetics tests** | `GET /api/v1/synthetics/tests` | `?page_size=100&page_number=0` |
| Synthetics search | `GET /api/v1/synthetics/tests/search` | `?text=env:prod type:api&count=50` |
| Trigger | `POST /api/v1/synthetics/tests/trigger` | `{"tests":[{"public_id":"abc-def-ghi"}]}` |
| Results | `GET /api/v1/synthetics/tests/{public_id}/results` | `?from_ts=&to_ts=` |
| **Dashboards** | `GET /api/v1/dashboard` | `?filter[shared]=false&filter[deleted]=false&count=100&start=0` |
| Dashboard | `GET/PUT /api/v1/dashboard/{id}` | `template_variables[]` |
| Dashboard lists | `GET /api/v2/dashboard/lists/manual/{id}/dashboards` | — |
| **Events (search)** | `POST /api/v2/events/search` | `{"filter":{"query":"source:nagios status:error","from":"now-1h","to":"now"},"page":{"limit":100}}` |
| Events (GET) | `GET /api/v2/events` | `?filter[query]=...&filter[from]=now-1h` |
| Events (v1) | `GET /api/v1/events` | `?start=<epoch_s>&end=<epoch_s>&sources=nagios&tags=env:prod&priority=normal` |
| Post event | `POST /api/v2/events` or `/api/v1/events` | `tags[]`, `aggregation_key`, `alert_type` |
| **DBM** | *(use metrics + logs-style APIs)* | `query=sum:postgresql.queries.count{...}` |
| **Containers** | `GET /api/v2/containers` | `?filter[tags]=kube_namespace:payments&group_by=kube_deployment&page[size]=100` |
| Container images | `GET /api/v2/container_images` | `?filter[tags]=short_image:nginx` |
| **Processes** | `GET /api/v2/processes` | `?search=nginx&tags=env:prod&from=<epoch_s>&to=<epoch_s>&page[limit]=100` |
| **SLOs** | `GET /api/v1/slo/search` | `?query=team:noc&page[size]=100` |
| **Service checks** | `POST /api/v1/check_run` | `check`, `host_name`, `status` (0–3), `tags[]` |
| **Users / roles** | `GET /api/v2/users` | `?filter=shayne&filter[status]=Active` |
| **Audit** | `POST /api/v2/audit/events/search` | `filter.query` event-platform syntax |
| **Validate key** | `GET /api/v1/validate` | — |


## 17.3 Pagination patterns


| Style | APIs | Params |
|---|---|---|
| Cursor | Logs, Spans, Events v2, Audit, RUM | `page[cursor]` from `meta.page.after` / `links.next` |
| Page number | NDM, Monitor search, Synthetics | `page[number]` + `page[size]` / `page` + `per_page` |
| Offset | Hosts, Dashboards | `start` + `count` |
| None / single | v1 query | — |


## 17.4 curl examples


```bash
export DD_SITE="api.datadoghq.com"

# Monitor search
curl -G "https://$DD_SITE/api/v1/monitor/search" \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  --data-urlencode 'query=status:alert tag:"env:prod"'

# Metric query
curl -G "https://$DD_SITE/api/v1/query" \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  --data-urlencode "from=$(($(date +%s)-3600))" \
  --data-urlencode "to=$(date +%s)" \
  --data-urlencode 'query=avg:system.cpu.user{env:prod} by {host}'

# Log search
curl -X POST "https://$DD_SITE/api/v2/logs/events/search" \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"query":"service:web status:error","from":"now-15m","to":"now"},"page":{"limit":50}}'

# NDM devices
curl -G "https://$DD_SITE/api/v2/ndm/devices" \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  --data-urlencode 'filter[tag]=device_namespace:default' \
  --data-urlencode 'page[size]=100'
```


## 17.5 PowerShell examples


```powershell
$site    = "api.datadoghq.com"
$headers = @{
  "DD-API-KEY"         = $env:DD_API_KEY
  "DD-APPLICATION-KEY" = $env:DD_APP_KEY
  "Content-Type"       = "application/json"
}

# Monitor search (URL-encode the query!)
$q = [uri]::EscapeDataString('status:alert tag:"env:prod"')
Invoke-RestMethod -Uri "https://$site/api/v1/monitor/search?query=$q" -Headers $headers

# Metric query
$to   = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
$from = $to - 3600
$mq   = [uri]::EscapeDataString('avg:system.cpu.user{env:prod} by {host}')
Invoke-RestMethod -Uri "https://$site/api/v1/query?from=$from&to=$to&query=$mq" -Headers $headers

# Log search
$body = @{
  filter = @{ query = "service:web status:error"; from = "now-15m"; to = "now" }
  page   = @{ limit = 50 }
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Method Post -Uri "https://$site/api/v2/logs/events/search" `
  -Headers $headers -Body $body

# Hosts filtered by tag
Invoke-RestMethod -Uri "https://$site/api/v1/hosts?filter=env:prod&count=1000" -Headers $headers

# Cursor pagination (logs)
$cursor = $null
do {
  $b = @{ filter = @{ query = "env:prod status:error"; from = "now-1h"; to = "now" }
          page   = @{ limit = 1000; cursor = $cursor } } | ConvertTo-Json -Depth 5
  $r = Invoke-RestMethod -Method Post -Uri "https://$site/api/v2/logs/events/search" -Headers $headers -Body $b
  $r.data | ForEach-Object { $_.attributes.message }
  $cursor = $r.meta.page.after
} while ($cursor)
```

## 17.6 Python client example


```python
from datadog_api_client import ApiClient, Configuration
from datadog_api_client.v1.api.monitors_api import MonitorsApi
from datadog_api_client.v2.api.logs_api import LogsApi
from datadog_api_client.v2.model.logs_list_request import LogsListRequest
from datadog_api_client.v2.model.logs_query_filter import LogsQueryFilter

cfg = Configuration()                     # reads DD_API_KEY / DD_APP_KEY / DD_SITE
with ApiClient(cfg) as c:
    mons = MonitorsApi(c).search_monitors(query='status:alert tag:"env:prod"')
    logs = LogsApi(c).list_logs(body=LogsListRequest(
        filter=LogsQueryFilter(query="service:web status:error",
                               _from="now-15m", to="now")))
```


## 17.7 API design rule


When automating Datadog:

``` text
1. Choose resource/API endpoint.
2. Read that endpoint's filter/query parameters.
3. Determine whether it embeds a Datadog search expression.
4. URL-encode GET parameters.
5. Use the correct Datadog site.
6. Handle pagination.
7. Handle rate limits.
8. Never assume a UI query is accepted by an unrelated API endpoint.
```

---

## 17.8 Encoding, pagination, and rate limits

### URL encoding

Every query sent as a GET parameter must be encoded. The characters that bite:

| Character | Encoded | Appears in |
|---|---|---|
| `@` | `%40` | Every event-platform attribute query |
| `:` | `%3A` | Every tag and facet query |
| space | `%20` | Boolean operators, phrases |
| `"` | `%22` | Quoted tags, phrases |
| `{` `}` | `%7B` `%7D` | Metric scopes |
| `[` `]` | `%5B` `%5D` | Ranges, `page[size]` |
| `,` | `%2C` | Multi-tag metric scopes |

POST-based search endpoints (logs, RUM, CI, spans) take the query in a JSON
body and need no encoding - prefer them for anything non-trivial, and for any
query long enough to hit a URL length limit.

### Pagination

| API generation | Mechanism |
|---|---|
| v1 (monitors, hosts, SLOs) | `page[size]` / `page[number]`, or `start` / `count` depending on endpoint |
| v2 event-platform (logs, spans, RUM, CI, audit) | Cursor: read `meta.page.after`, send it back as `page[cursor]` |
| DDSQL | Async `query_id`, polled at a fetch endpoint; page with SQL `OFFSET`/`LIMIT` |

Cursor pagination does not accept a page number. Any script that tries to "jump
to page 5" against a v2 endpoint is silently wrong.

### Rate limits

Responses carry `X-RateLimit-Limit`, `X-RateLimit-Period`, `X-RateLimit-Remaining`,
and `X-RateLimit-Reset`. Limits differ per endpoint family - log search, metric
submission, and monitor writes each have their own budget.

Practical rules for bulk automation:

- Read `X-RateLimit-Remaining` and back off *before* you get a 429, not after.
- On 429, honor `X-RateLimit-Reset` rather than a fixed sleep.
- Batch reads (one list call plus local filtering) instead of one call per
  object; a "get each monitor by id" loop is the usual cause of a rate-limit
  incident during a migration.
- Make writes idempotent: look up, diff, and `PATCH` only what changed. A retry
  after a 429 should be a no-op if the previous attempt actually succeeded.

### Permissions

A 403 can mean a missing scope *or* an unlicensed product. `"Failed permission
authorization checks"` in the response body confirms it is permissions. An
application key can never grant more than its owner's role allows, so a key
created by a restricted user will fail in ways that look like API bugs.


---


# 18. Time syntax reference


| Context | Format | Example |
|---|---|---|
| v1 metric/event API | Epoch **seconds** | `1726500000` |
| v2 event-platform APIs | Relative, ISO 8601, or epoch **ms** | `now-15m`, `2026-09-17T12:00:00Z`, `1726500000000` |
| Dashboard URLs | Epoch **ms** | `from_ts=1726500000000` |
| Relative units | `s`, `m`, `h`, `d`, `w`, `mo` | `now-7d` |
| Monitor windows (metric) | `last_Xm/h/d/w` | `last_15m` |
| Monitor windows (log/event/trace) | `last("X")` | `last("15m")` |
| Span duration facet | ns by default; units allowed | `@duration:>500ms` |
| Log `@duration` (standard attribute) | nanoseconds | `@duration:>1000000000` = 1s |


---


# 19. Cross-product examples

Assume a server has:

``` text
env:prod
app:abc
costcenter:eoc
site:bos
team:noc
```

## Goal: CPU for ABC production hosts

### Metrics

``` text
avg:system.cpu.user{env:prod AND app:abc} by {host}
```

### Metric monitor

``` text
avg(last_5m):avg:system.cpu.user{env:prod AND app:abc} by {host} > 90
```

### Dashboard

Use variables:

``` text
$env
$app
```

and apply them to the metric widget.

### Host List

Filter using the host tags:

``` text
env:prod
app:abc
```

### Logs

``` text
env:prod AND app:abc
```

### APM

``` text
env:prod AND service:abc
```

assuming `service:abc` is your APM service tag.

### Monitor List: find monitors tagged for ABC

``` text
tag:app:abc
```

### Monitor List: find monitors whose scope contains ABC

``` text
scope:app:abc
```

Those last two are **not interchangeable**.

------------------------------------------------------------------------


---

# 20. Common mistakes and hard-won lessons


## 20.1 Confusing monitor tags with metric tags

Monitor:

``` text
tag:app:abc
```

does not mean the monitor evaluates telemetry tagged `app:abc`.

Monitor metadata and telemetry scope are separate concepts.

## 20.2 Using log syntax in metric filters

Log/event exclusion:

``` text
-version:beta
```

Metric functional syntax:

``` text
NOT version:beta
```

Do not assume they are interchangeable.

## 20.3 Mixing metric Boolean styles

Avoid:

``` text
env:prod AND !region:us-east
```

Use:

``` text
env:prod AND NOT region:us-east
```

## 20.4 Forgetting `@` on event attributes

Wrong for a custom log/span attribute:

``` text
http.status_code:500
```

Usually correct:

``` text
@http.status_code:500
```

Reserved fields such as `service`, `host`, `source`, `status`, and
certain APM fields are exceptions.

## 20.5 Quoting a wildcard

Often:

``` text
service:web*
```

means wildcard.

Whereas:

``` text
service:"web*"
```

may treat `*` literally.

## 20.6 Treating a facet and attribute as the same concept

A field can often be searched as an attribute without first creating a
facet, but facets provide UI navigation, aggregation, and, for some
numerical operations, are required.

## 20.7 Using `by` as a filter

This:

``` text
by {host}
```

does not filter hosts.

It splits the selected data into one series/group per host.

## 20.8 Assuming `{*}` means the same as no query constraints everywhere

`{*}` is metric syntax.

Do not paste it into Log Explorer or Trace Explorer.

## 20.9 Assuming every UI has identical wildcard behavior

It does not.

Check the product-specific search documentation.

## 20.10 Guessing tag availability

A tag only helps if it is actually attached to the telemetry/resource
you're querying.

A host having:

``` text
app:abc
```

does not automatically guarantee every log, span, device, Synthetic
result, or custom metric has the same tag.

------------------------------------------------------------------------


## 20.11 Additional gotchas


1. **Negation differs by family.** Metrics use `!env:dev`; logs/APM/events use `-env:dev`. `NOT` works in both.
2. **`@` only exists in event-platform searches.** Metric scopes are tags only.
3. **Facets are case-sensitive; full-text is not.** Normalize tag values to lowercase at submission — the Agent lowercases, but API/DogStatsD submissions may not, causing fragmentation (`env:Prod` ≠ `env:prod`).
4. **Monitor Notification Rules match monitor tags**, not host or metric tags. A monitor without `app:<acronym>` won't match an `app:` rule.
5. **`datadog.agent.running` is unreliable for host-down.** Use the `datadog.agent.up` service check or the native Host monitor.
6. **`count_nonzero()` / `count_not_null()` go in `formulas`** in the v1 dashboard API, not in the `query` string.
7. **v1 dashboard API** doesn't support `reflow_type` or the `group` widget, and auto-flows widgets without layout objects.
8. **A 403 can mean missing scope *or* unlicensed product.** "Failed permission authorization checks" in the body confirms a permissions issue. A key can never exceed its owner's role.
9. **Comma = AND in metric scopes.** `{env:prod,env:staging}` matches nothing; use `{env IN (prod,staging)}`.
10. **Wildcard in quotes is literal** — `"web*"` won't expand.
11. **Log `@duration` is nanoseconds**; span duration accepts units.
12. **`.as_count()` changes rollup math.** For counters in monitors, set it explicitly and prefer `sum` aggregation.
13. **Legacy vs current Events syntax** — `sources:` / `tags:` (v1) vs `source:` / tag-as-facet (v2). Don't mix them.
14. **Docs can be wrong.** Validate metric names, tag keys, and field placements against Metrics Explorer or a live API response before scaling automation. Test one monitor/rule first, then bulk-apply with `--dry-run`.
15. **Boolean operators must be uppercase.** `service:web and status:error` searches for the literal word `and`.
16. **`status:` means three different things.** Severity in security signals, log level in Logs, monitor state in the Monitor List.
17. **`?` is a single-character wildcard**, not a typo guard. It is the cleanest way to match a value containing a space or colon.
18. **Event-level selection comes before filtering.** RUM (`@type:`), CI Visibility (pipeline/stage/job/command), and Error Tracking (issue vs. error) all return different result sets for the same filter string.
19. **SLO monitors reference SLO IDs.** Deleting and re-creating an SLO orphans every error-budget and burn-rate monitor attached to it.
20. **Removing a tag from a metric's tag configuration silently breaks group-by.** The metric keeps reporting; the dimension disappears, and every monitor grouped by it stops alerting.
21. **Live Tail is not search.** It streams pre-index, so it shows logs that exclusion filters will drop and that no query will ever return.
22. **Saved views carry more than the query** - columns, sort order, time range, and facet state. A saved view pinned to an absolute time range is a snapshot, not a live view.


---


# 21. Recommended enterprise conventions

For a large Datadog environment, establish a predictable tag vocabulary.

## 21.1 Core tags

Recommended baseline:

``` text
env:<environment>
service:<service>
version:<version>
team:<team>
app:<application>
costcenter:<cost-center>
site:<site>
region:<region>
role:<role>
criticality:<level>
```

Example:

``` text
env:prod
service:billing-api
version:4.2.1
team:payments
app:abc
costcenter:eoc
site:bos
region:us-east
role:application
criticality:1
```

## 21.2 Avoid spaces

Prefer:

``` text
app:billing_portal
```

or:

``` text
app:billing-portal
```

rather than:

``` text
app:Billing Portal
```

Consistent machine-friendly tags make metric filters, APIs, monitors,
automation, and dashboards much easier to maintain.

## 21.3 Normalize case

Prefer:

``` text
env:prod
```

not a mixture of:

``` text
env:Prod
env:PROD
env:production
env:Production
```

Choose a canonical vocabulary and enforce it.

## 21.4 Separate identity from presentation

Machine tag:

``` text
app:abc
```

Display name:

``` text
Accounts Billing and Collections
```

Do not force long human descriptions into core operational tag values.

## 21.5 Use tags consistently across telemetry

The greatest value appears when:

``` text
env:prod
service:billing-api
app:abc
team:payments
```

can correlate:

``` text
metrics
logs
traces
hosts
containers
monitors
events
dashboards
```

------------------------------------------------------------------------


---


# 22. Quick-reference recipes

## Find alerting monitors

``` text
status:Alert
```

## Find muted monitors

``` text
muted:true
```

## Find metric monitors

``` text
type:metric
```

## Find monitors tagged for ABC

``` text
tag:app:abc
```

## Find monitors whose telemetry scope includes ABC

``` text
scope:app:abc
```

## CPU for production

``` text
avg:system.cpu.user{env:prod} by {host}
```

## CPU for three applications

``` text
avg:system.cpu.user{
  env:prod AND app IN (abc,xyz,mno)
} by {host}
```

## Exclude lab systems

``` text
avg:system.cpu.user{
  costcenter:eoc AND NOT env:lab
} by {host}
```

## Error logs

``` text
env:prod AND status:error
```

## HTTP 5xx logs

``` text
env:prod AND @http.status_code:[500 TO 599]
```

## Logs from an IP network

``` text
CIDR(@network.client.ip,10.0.0.0/8)
```

## APM errors for service

``` text
service:checkout AND env:prod AND status:error
```

## APM custom attribute

``` text
service:checkout AND @http.status_code:500
```

## Events from a source

``` text
source:github AND status:error
```

## DBM query samples

``` text
dbm_type:activity service:orders-db env:prod
```

## Containers containing Java but not Elasticsearch

``` text
java NOT elasticsearch
```

## Synthetic API tests

``` text
type:api AND env:prod
```

## Dashboard variable

``` text
$app
```

## Dashboard variable raw value

``` text
$app.value
```

------------------------------------------------------------------------


---


# 23. Official references

The syntax in this guide is based on Datadog's product documentation.
Because Datadog evolves quickly and features can differ by site, product
entitlement, and Gov/Federal environment, verify unusual or high-impact
production queries against the current docs.

-   Getting Started with Search\
    https://docs.datadoghq.com/getting_started/search/

-   Getting Started with Tags\
    https://docs.datadoghq.com/getting_started/tagging/

-   Using Tags\
    https://docs.datadoghq.com/getting_started/tagging/using_tags/

-   Monitor Search\
    https://docs.datadoghq.com/monitors/manage/search/

-   Metric Advanced Filtering\
    https://docs.datadoghq.com/metrics/advanced-filtering/

-   Host List\
    https://docs.datadoghq.com/infrastructure/list/

-   Log Search Syntax\
    https://docs.datadoghq.com/logs/explorer/search_syntax/

-   APM Trace Explorer Query Syntax\
    https://docs.datadoghq.com/tracing/trace_explorer/query_syntax/

-   Synthetic Results Explorer Search Syntax\
    https://docs.datadoghq.com/synthetics/explore/results_explorer/search_syntax/

-   Search and Manage Synthetic Tests\
    https://docs.datadoghq.com/synthetics/explore/

-   Dashboard Template Variables\
    https://docs.datadoghq.com/dashboards/template_variables/

-   Events Explorer Search Syntax\
    https://docs.datadoghq.com/events/explorer/searching/

-   DBM Query Samples\
    https://docs.datadoghq.com/database_monitoring/query_samples/

-   Containers Explorer\
    https://docs.datadoghq.com/containers/monitoring/containers_explorer/

-   Monitor API\
    https://docs.datadoghq.com/api/latest/monitors/

-   Hosts API\
    https://docs.datadoghq.com/api/latest/hosts/

-   Metrics API\
    https://docs.datadoghq.com/api/latest/metrics/

### Part II surfaces

-   RUM Explorer Search Syntax\
    https://docs.datadoghq.com/real_user_monitoring/explorer/search_syntax/

-   RUM Search Events API\
    https://docs.datadoghq.com/api/latest/rum/search-rum-events/

-   Error Tracking Explorer\
    https://docs.datadoghq.com/error_tracking/explorer

-   CI Visibility Explorer Search Syntax\
    https://docs.datadoghq.com/continuous_integration/explorer/search_syntax/

-   CI/CD and Test Monitors\
    https://docs.datadoghq.com/monitors/types/ci/

-   Investigate Security Signals\
    https://docs.datadoghq.com/security/cloud_siem/triage_and_investigate/investigate_security_signals/

-   Search for SLOs (API)\
    https://docs.datadoghq.com/api/latest/service-level-objectives/search-for-slos/

-   Audit Trail\
    https://docs.datadoghq.com/account_management/audit_trail/

-   DDSQL API\
    https://docs.datadoghq.com/api/latest/ddsql/

-   DDSQL Data Directory\
    https://docs.datadoghq.com/ddsql_reference/data_directory/

-   Tag Policies API (preview)\
    https://docs.datadoghq.com/api/latest/tag-policies/

------------------------------------------------------------------------


---

# 24. One-page decision tree


``` text
DATADOG QUERY / FILTER DECISION TREE
====================================

What are you searching?

├── Monitor definitions
│   └── Monitor List syntax
│       ├── status:Alert
│       ├── tag:app:abc
│       └── scope:env:prod
│
├── Metric timeseries
│   └── Metric syntax
│       └── avg:metric{env:prod AND app:abc} by {host}
│
├── Logs / APM / Events / event-style records
│   └── Event search syntax
│       ├── service:web AND env:prod
│       ├── @attribute:value
│       └── -version:beta
│
├── Hosts / Devices / Containers
│   └── Inventory search + facets + tags
│
├── Dashboard
│   └── Template variables + underlying query language
│       ├── $env
│       ├── $app
│       └── $app.value
│
└── API
    └── Endpoint-specific parameters
        └── May embed one of the query languages above
```

## Golden rule

> **First identify the Datadog data surface. Then choose its query
> language.**

If a query works in Metrics Explorer, that does not guarantee it works
in Log Explorer.\
If a search works in Manage Monitors, that does not mean it is a valid
monitor telemetry query.\
If a tag exists on a host, that does not guarantee it exists on every
related metric, log, span, container, device, or Synthetic result.

That small mental model turns Datadog filtering from a syntax jungle
into a map.


---

# Part II — Extended product surfaces

Sections 1–24 cover the surfaces most NOC and platform teams touch daily.
Part II covers the remaining surfaces that use their own facets, their own
attribute namespaces, or — in the case of DDSQL — an entirely different
grammar.

Everything in Part I still applies: **identify the data surface first**, then
choose its query language. The per-surface "verify before you automate" rule
applies even more strongly here, because several of these products are newer,
are entitlement-gated, and are not available on every Datadog site (notably
`app.ddog-gov.com` / `us2.ddog-gov.com`).

| Section | Surface | Family |
|---|---|---|
| 25 | RUM and Session Replay | Event-platform |
| 26 | Error Tracking | Event-platform (issue-level) |
| 27 | CI Visibility and Test Optimization | Event-platform (log syntax) |
| 28 | Security signals (Cloud SIEM, Workload Protection, AAP) | Event-platform |
| 29 | SLOs | Faceted object search + metric/monitor queries |
| 30 | Audit Trail | Event-platform |
| 31 | DDSQL | SQL |
| 32 | Notebooks, saved views, shareable URLs | Host product's family |
| 33 | Tag governance surfaces | API/object |
| 34 | Terraform and query-as-code | Embeds all of the above |
| 35 | Other surfaces in brief | Mixed |
| 36 | Part II decision addendum | — |

---

# 25. Real User Monitoring (RUM) and Session Replay

RUM uses event-platform search, but with one twist that trips people up: the
**event type selector changes which attributes exist**. A query that works on
views will silently return nothing on sessions.

## 25.1 Event types

``` text
@type:session
@type:view
@type:action
@type:error
@type:resource
@type:long_task
@type:vital
```

Rule of thumb:

- **Session** — one user visit. Aggregates (`@session.error.count`) live here.
- **View** — one page/screen. Performance attributes (`@view.loading_time`) live here.
- **Action** — clicks, taps, custom actions.
- **Resource** — network requests issued by the page (XHR, fetch, assets).
- **Error** — front-end errors, crashes.
- **Long task / vital** — main-thread blocking and Core Web Vitals.

## 25.2 Common attributes

``` text
@application.id:<application-uuid>
@session.type:user
@session.type:synthetics
@session.error.count:>5
@session.error.count:[3 TO 10]
@view.url_path:"/department/sofas"
@view.loading_time:[1s TO 3s]
@usr.id:12345
@usr.email:someone@example.gov
@error.source:console
@resource.status_code:[500 TO 599]
```

Exclude synthetic browser-test traffic from user-facing numbers:

``` text
@type:view -@session.type:synthetics env:prod
```

## 25.3 Escaping, wildcards, and ranges

| Want | Syntax |
|---|---|
| Exact path | `@view.url_path:"/department/sofas"` |
| Path prefix (slashes escaped) | `@view.url_path:\/department\/sofas\/*` |
| URL prefix | `@http.url:https:\/\/*` |
| Facet name containing a space | `@user.first\ name:myvalue` |
| Numeric comparison | `@session.error.count:>5` |
| Numeric range | `@session.error.count:[3 TO 10]` |
| Duration range with units | `@view.loading_time:[1s TO 3s]` |

Wildcards are only wildcards **outside** double quotes. `@view.url_path:"*checkout*"`
matches the literal string `*checkout*`.

## 25.4 RUM-based metrics

RUM-based metrics turn RUM events into a timeseries you can graph, monitor, and
retain long-term:

``` text
event_type:  session | view | action | error | resource | long_task | vital
filter:      service:web* AND @http.status_code:[200 TO 299]
compute:     count  |  distribution on a path such as @duration
group_by:    path @http.status_code  ->  tag_name status_code
```

Notes:

- `group_by.tag_name` is the **tag key the metric will carry**. It does not have
  to match the attribute path, and renaming it later breaks existing queries.
- Distribution metrics can include percentiles; that increases custom metric
  volume, so decide deliberately.

## 25.5 RUM search API

``` json
POST /api/v2/rum/events/search
{
  "filter": {
    "from": "now-15m",
    "query": "@type:session AND @session.type:user",
    "to": "now"
  },
  "options": { "timezone": "GMT" },
  "page": { "limit": 25 },
  "sort": "timestamp"
}
```

`sort` accepts `timestamp` or `-timestamp`. Paginate with the cursor returned in
`meta.page.after`, passed back as `page.cursor`.

## 25.6 RUM monitors

RUM monitors follow the log-monitor shape: a RUM search query, an event level
(session/view/error/…), a count or measure, optional group-by, and a threshold.
Group by `@application.id` or `service` so notification routing can distinguish
front-end applications.

---

# 26. Error Tracking

Error Tracking is not a separate search language — it is an **aggregation layer**
over errors already present in Logs, APM, and RUM. The unit you search is an
**issue**, not an individual error event.

## 26.1 How issues are formed

Datadog computes a fingerprint for each error from attributes such as the error
type, the error message, and the stack trace; errors sharing a fingerprint are
grouped into one issue. This matters operationally: a badly formatted error
message that embeds a unique ID (a request ID, a hostname, a timestamp) can
fragment one bug into thousands of issues.

## 26.2 Issue markers

| Marker | Meaning |
|---|---|
| **New** | First seen less than two days ago and still in the *For Review* state |
| **Regression** | Was resolved, then occurred again in a newer version |
| **Crash** | The error crashed the application |
| **Suspected cause** | Datadog has inferred a likely originating change |

Issue states drive triage workflow (for review / resolved / ignored). The state
is exposed as a facet in the explorer — confirm the exact facet key in the facet
panel before scripting against it, because it differs from the human-readable
label.

## 26.3 Searching

Search terms are the attributes of the underlying error source, so the usable
attribute set depends on where the errors came from:

``` text
service:checkout env:prod @error.type:TimeoutError
service:checkout @error.message:*connection\ refused*
@error.source:logger version:2.4.1
```

Sorting options — Relevance, Count, Newest, Impacted Sessions — change which
issues surface first but not which issues match. For dashboards and reports,
sort by Count; for triage, Relevance.

## 26.4 Operational advice

- Version tagging is what makes Regression detection work. Without a consistent
  `version:` tag, Error Tracking cannot tell a regression from a recurrence.
- Error Tracking monitors alert on issue volume or on new issues appearing;
  route them by `service` and `team`, not by issue.

---

# 27. CI Visibility and Test Optimization

CI Visibility pipeline and test events are searched with **log search syntax** —
the same grammar as Section 9, including `-` negation, `@` attributes, ranges,
and the "wildcards only outside quotes" rule.

## 27.1 Pipeline attributes

``` text
@ci.status:error
@ci.status:(success OR error OR canceled)
@ci.pipeline.name:"deploy-prod"
@ci.pipeline.id:<id>
@ci.stage.name:build
@ci.job.name:unit-tests
@git.branch:main
@git.repository.id_v2:<repo>
```

## 27.2 Event levels

CI pipeline data is hierarchical: **pipeline → stage → job → command**. The
level is a first-class selector, not just another filter:

- In the explorer, the level is chosen in the UI next to the search bar.
- In a CI monitor, the level is a separate configuration field.
- Some views expose a level facet; confirm its key in the facet panel before
  hard-coding it into automation.

Searching at the wrong level is the single most common CI Visibility mistake:
counting `@ci.status:error` at job level and at pipeline level gives very
different numbers for the same failure.

## 27.3 Test attributes

``` text
@test.status:fail
@test.name:"test_login_redirect"
@test.suite:auth
@test.service:web-api
@test.is_flaky:true
```

## 27.4 CI/CD and Test monitors

- Alert on a **count** of matching events, or on a **measure** (a quantitative
  facet) with an aggregation: `min`, `avg`, `sum`, `median`, `pc75`, `pc90`,
  `pc95`, `pc98`, `pc99`, `max`.
- Group by up to **four** facets; each group alerts independently. Grouping by
  `@ci.pipeline.name` gives one alert per pipeline.
- Rates are built with formulas — for example, failed pipeline events (query a:
  `@ci.status:error`) over total pipeline events (query b: no filter), with
  formula `a / b`, grouped by `@ci.pipeline.name`.
- There is a default limit of 1000 CI/CD and Test monitors per account.

## 27.5 APIs

``` text
POST /api/v2/ci/pipelines/events/search
POST /api/v2/ci/tests/events/search
GET  /api/v2/ci/pipelines/events?filter[query]=...
GET  /api/v2/ci/tests/events?filter[query]=...
```

These take a log search query and paginate the same way the Logs API does
(cursor in `meta.page.after`).

---

# 28. Security signals (Cloud SIEM, Workload Protection, App and API Protection)

All of Datadog's signal explorers share one search model. The signal is an
event-platform record; detection rules that *produce* signals are written in the
log search syntax of whatever data they analyze.

## 28.1 Signal search

``` text
status:(critical OR high)
status:(high OR critical OR medium) @workflow.triage.state:(open OR under_review)
@workflow.rule.name:"Excessive login failures"
@workflow.rule.type:"Application Security"
```

| Attribute | Values |
|---|---|
| `status` | `info`, `low`, `medium`, `high`, `critical` |
| `@workflow.triage.state` | `open`, `under_review`, `archived` |
| `@workflow.rule.name` | Detection rule name |
| `@workflow.rule.type` | Product/rule family |

Note the vocabulary collision: `status` means **severity** for security signals,
but **log level** in Log Explorer and **monitor state** in the Monitor List.
Three surfaces, one word, three meanings.

## 28.2 Detection rules and suppressions

A detection rule carries one or more queries in log search syntax plus a
grouping and a threshold. Suppressions are separate objects with two queries:

``` text
rule_query:        ruleId:abc-def-ghi OR ruleId:jkl-mno-pqr
suppression_query: @usr.email:(svc-account@example.gov) AND @network.client.ip:10.0.0.0/8
```

Suppression is the correct tool for approved service accounts and known scanner
hosts — muting the signal downstream hides the finding instead of scoping it.

## 28.3 Notification rules

Security signal notification rules are a **different object** from monitor
notification rules (Section 6.8) with a different filter vocabulary. Rules for
monitors will never match signals, and vice versa.

---

# 29. Service Level Objectives

SLOs involve three distinct query surfaces at once, which is why they confuse
people: the SLO **object search**, the **SLI query** inside the SLO, and the
**error-budget monitor** built on top.

## 29.1 SLO list search

The Manage SLOs list is a faceted object search over SLO definitions:

``` text
service:checkout
team:noc
slo_type:metric
env:prod
"Checkout Availability"
```

Search API:

``` text
GET /api/v1/slo/search?query=service:checkout&page[size]=25&page[number]=0&include_facets=true
```

`include_facets=true` returns the facet breakdown Datadog itself uses:
`all_tags`, `creator_name`, `env_tags`, `service_tags`, `slo_type`, `target`,
`team_tags`, `timeframe`. That response is the cheapest way to audit which tag
keys are actually present on SLOs before running a governance sweep. The
endpoint requires the `slos_read` permission.

## 29.2 SLI queries by SLO type

| SLO type | What the query looks like |
|---|---|
| **Metric-based** | Two metric queries — numerator (good events) and denominator (total events), both usually `.as_count()` |
| **Monitor-based** | References monitor IDs, not a query; the monitor's own query is the SLI |
| **Time slice** | A metric query plus a condition evaluated per slice, e.g. `avg:system.cpu.user{app:abc} < 80` |

Metric-based example:

``` text
numerator:   sum:trace.http.request.hits{env:prod,service:checkout}.as_count() - sum:trace.http.request.errors{env:prod,service:checkout}.as_count()
denominator: sum:trace.http.request.hits{env:prod,service:checkout}.as_count()
```

## 29.3 Error budget monitors

Error-budget and burn-rate monitors reference the SLO by ID rather than
re-stating the query:

``` text
error_budget("<slo_id>").over("7d") > 75
burn_rate("<slo_id>").over("1h").long_window("24h") > 14.4
```

Because they reference an ID, renaming or re-creating an SLO silently orphans
its monitors. Treat SLO IDs as durable identifiers in any automation.

## 29.4 Tagging

SLO tags are their own tag set — they are not inherited from the metrics,
monitors, or services inside the SLO. A tag governance policy that checks hosts
and monitors but not SLOs will report clean while SLO routing is broken.

---

# 30. Audit Trail

Audit Trail records configuration and access events for the Datadog org itself.
For regulated environments this is the surface that answers "who changed this
monitor, and when" — and it is searched with ordinary event-platform syntax.

## 30.1 Core attributes

| Attribute | Meaning |
|---|---|
| `@evt.name` | Product area — `Monitors`, `Metrics`, `Log Management`, `APM`, `Synthetics Monitoring`, `Real User Monitoring`, `Sensitive Data Scanner`, `api_key` |
| `@asset.type` | Object type — `monitor`, `index`, `metric`, `synthetics_test`, `retention_filter`, `custom_metrics` |
| `@action` | `created`, `modified`, `deleted` |
| `@usr.email`, `@usr.id`, `@usr.name` | Actor |
| `@metadata.api_key.id` | API key used (key id, never the key itself) |
| `@http.method`, `@http.url_details.path` | API call shape |
| `@network.client.ip`, `@network.client.geoip.country.name` | Source of the change |

## 30.2 Useful queries

``` text
# Monitor changes in the last day
@evt.name:Monitors @action:(created OR modified OR deleted)

# Metric tag configuration changes
@evt.name:Metrics @asset.type:metric @action:(created OR modified)

# Log index or retention changes
@evt.name:"Log Management" @asset.type:index

# Synthetic tests created or deleted
@evt.name:"Synthetics Monitoring" @asset.type:synthetics_test @action:(created OR deleted)

# Everything done with a specific API key
@metadata.api_key.id:<key-id>

# Changes made by anything other than your automation accounts
@action:modified -@usr.email:(automation@example.gov OR terraform@example.gov)
```

## 30.3 Operational uses

- **Key rotation verification** — after rotating a key, confirm the old key id
  stops appearing in `@metadata.api_key.id`.
- **Change attribution during migration** — when a monitor's behavior changes
  unexpectedly, the diff tab on an audit event shows the before/after config.
- **Drift detection** — alert on `@action:modified` for objects that should only
  ever be changed by automation.

Audit events can be exported to a log index or queried through DDSQL
(Section 31), which is usually easier for scheduled compliance reporting.

---

# 31. DDSQL — SQL over Datadog data

DDSQL is the fourth syntax family and the newest. It queries Datadog's data
catalog with SQL rather than a search bar, which makes it the right tool for
joins, aggregates, and inventory reporting that the explorers cannot express.

## 31.1 Shape

``` sql
SELECT COUNT(*) FROM dd.hosts;

SELECT cloud_provider, COUNT(*) AS hosts
FROM dd.hosts
GROUP BY cloud_provider
ORDER BY hosts DESC;
```

Datasets live under the `dd.` namespace (`dd.hosts`, `dd.audit`, and others in
the DDSQL Data Directory). Host tags arrive as an `hstore`-style key/value
column rather than a flat string, so tag predicates look like column access,
not like `{env:prod}`.

## 31.2 Table functions embed event-platform search

Some datasets are **polymorphic table functions**: they require parameters, and
one of those parameters is an ordinary event-platform query string. This is the
one place where two syntax families legitimately appear in the same statement:

``` sql
SELECT * FROM dd.audit(
  columns => ARRAY['timestamp','@usr.email','@evt.name','@metadata.api_key.id'],
  filter  => '@metadata.api_key.id:* AND @evt.name:api_key',
  from_timestamp => now() - interval '24 hours',
  to_timestamp   => now()
) AS (ts TIMESTAMP, user_email VARCHAR, event_name VARCHAR, api_key_id VARCHAR);
```

The `filter` string follows Section 30 rules; the surrounding statement follows
SQL rules. Do not try to express the filter as a SQL `WHERE` clause — the
function needs it up front.

## 31.3 API

``` text
POST /api/v2/ddsql/query/tabular          # submit
POST /api/v2/ddsql/query/tabular/fetch    # poll with query_id
```

Queries are dispatched asynchronously. The submit call returns either
`state: completed` with the result set inlined, or `state: running` plus an
opaque `query_id`; poll the fetch endpoint with that id until it completes.
Results are **column-major** — a list of columns each holding a list of values,
not a list of row objects — so client code that assumes row dictionaries needs a
transposition step.

Other practical notes:

- Row limit defaults to 5000; page with SQL `OFFSET n LIMIT m` rather than
  client-side slicing.
- Timestamps and dates come back as Unix milliseconds; a `DATE` resolves to
  midnight UTC.
- Bound time with the table function's own `from_timestamp` / `to_timestamp`
  parameters where they exist, rather than a `WHERE` clause on a time column.

## 31.4 When to reach for DDSQL

| Question | Better surface |
|---|---|
| "How many hosts per cloud provider, per tag value?" | DDSQL |
| "Which hosts are missing the `app` tag?" | DDSQL, or the Hosts API |
| "Show me CPU for these hosts over time" | Metrics (Section 8) |
| "What changed last night?" | Audit Trail, or DDSQL over `dd.audit` for scheduled reports |

---

# 32. Notebooks, saved views, and shareable URLs

## 32.1 Notebooks

Notebook cells embed whichever query language the cell's data source uses — a
metric cell takes metric syntax, a log cell takes log search syntax. Notebooks
also support log **full-text search** (`*:term`), which is one of the few places
outside the Log Explorer where it works, along with monitors and dashboards.

## 32.2 Saved views

A saved view stores more than the query. In the explorers it captures the search
query, the visible columns, the sort order, the time range, and the facet panel
state. Two consequences:

- A saved view with an absolute time range is a snapshot, not a live view.
- Sharing a saved view shares the columns and sort, which is often the actual
  knowledge being transferred — the query alone is rarely enough.

## 32.3 URLs as an API

Explorer URLs carry the query as a parameter, which makes them scriptable
deep links for runbooks and notification messages:

``` text
/logs?query=service%3Acheckout%20status%3Aerror&from_ts=...&to_ts=...&live=true
/monitors/manage?q=status%3AAlert%20tag%3A%22app%3Aabc%22
/metric/explorer?exp_metric=system.cpu.user&exp_scope=env%3Aprod
```

Rules that save time:

- URL-encode the whole query: `@` becomes `%40`, `:` becomes `%3A`, space
  becomes `%20`, `{` `}` `[` `]` `"` all need encoding.
- Dashboard and explorer URLs use epoch **milliseconds** for time bounds
  (Section 18).
- A deep link in a monitor message is worth more than a paragraph of
  explanation; put the exact filtered view the responder needs in the runbook
  field.

---

# 33. Tag governance surfaces

Filtering is only as good as the tags underneath it. These are the surfaces that
govern the tags themselves.

## 33.1 Tag Policies

Tag Policies define which tag values are accepted for a given tag key, scoped to
a telemetry source (logs, spans, metrics, and so on). Policies are either:

- **blocking** — telemetry that does not match is rejected, or
- **surfacing** — non-matching data is highlighted but still ingested.

Each policy reports a **compliance score** derived from how much recent
telemetry adheres to it.

``` text
GET    /api/v2/tag_policies            # list; include=score for compliance
POST   /api/v2/tag_policies            # create
GET    /api/v2/tag_policies/{policy_id}
DELETE /api/v2/tag_policies/{policy_id}   # soft delete by default
GET    /api/v2/tag_policies/{policy_id}/score
```

Caveats:

- This API is in preview; the path has already changed once (hyphen to
  underscore). Pin nothing without checking a live response first.
- `blocking` mode drops data. Run a policy in `surfacing` mode long enough to
  see its score stabilize before switching it.
- Requires `telemetry_rules_read` / `metrics_read` (read) permissions.

## 33.2 Metric tag configuration

Metric-level tag configuration controls which tags are queryable on a custom
metric. Removing a tag from the configuration makes every existing query and
monitor that groups by that tag return nothing — the metric still exists, the
dimension does not. Audit tag configuration changes through Audit Trail
(`@evt.name:Metrics @asset.type:metric`).

## 33.3 Restriction policies

Restriction policies bind roles to individual resources (a dashboard, a monitor,
a notebook) rather than to a whole product. They are the reason a query can work
for you and return nothing for a teammate. When a user reports missing data,
check restriction policies and role scopes before debugging the query itself.

## 33.4 A governance checklist that survives an audit

1. One canonical tag key list, lowercase, documented.
2. Normalize values **before** submission, on every path — Agent, API, and
   DogStatsD — because only the Agent path lowercases reliably.
3. Check tag presence on every object type, not just hosts: monitors,
   synthetics, SLOs, dashboards, log pipelines, APM services.
4. Report compliance as a trend, not a snapshot; a single-point score hides
   whether a migration is converging.
5. Gate automation on a dry run that prints per-field changes.

---

# 34. Terraform and query-as-code

Query strings are **opaque to Terraform**. The provider will happily apply a
syntactically valid monitor whose query matches nothing.

## 34.1 Interpolation collisions

Terraform interpolates `${...}`. Datadog monitor messages use `{{...}}` for
template variables, and Datadog dashboards use `$var` for template variables.
The failure modes:

``` hcl
# WRONG - Terraform tries to interpolate
message = "CPU high on ${host.name}"

# RIGHT - escape the dollar sign for Terraform
message = "CPU high on $${host.name}"

# Monitor template variables use double braces and are safe
message = "CPU high on {{host.name}} for {{app.name}}"
```

## 34.2 Drift and diff noise

- Datadog normalizes some queries server-side (whitespace, operator spelling).
  A query written as `{env:prod AND app:abc}` may come back as
  `{env:prod,app:abc}`, producing perpetual diffs. Write queries in the form the
  API returns them.
- `validate = true` on a monitor resource asks Datadog to validate the query at
  plan time. Use it; it converts a silent no-match into a plan failure.
- Monitor `restricted_roles` and restriction policies can be managed in two
  places at once. Pick one and stay there.

## 34.3 Generated over handwritten

For large estates, generate monitor and dashboard definitions from a single
source of truth (a CSV or YAML inventory) rather than maintaining hundreds of
near-identical blocks. The generator owns naming, tagging, and query shape;
the state file owns lifecycle. The same argument applies to Python automation
against the API directly: keep the query construction in one function so a
syntax fix lands everywhere at once.

---

# 35. Other surfaces in brief

These surfaces reuse a family already covered; only their vocabulary differs.
Confirm attribute names in the product's facet panel before automating.

| Surface | Family | Notes |
|---|---|---|
| **Incident Management** | Faceted object search | Facets such as state, severity, commander, and incident tags. Incident search is separate from the monitors and signals that created the incident. |
| **Case Management** | Faceted object search | Cases group signals or issues for triage; case attributes are not signal attributes. |
| **Cloud Cost Management** | Metric-style with cost-specific tag keys | Cost data is queried like metrics, scoped by cloud and allocation tags. Tag coverage on cloud resources determines whether cost can be split by team or application at all. |
| **Software Catalog / Teams** | Faceted object search | Entity-level search over services and their owners; `team` here is a catalog concept, not automatically the `team:` tag on telemetry. |
| **Workflow Automation** | Trigger filters use the source product's syntax | A monitor-triggered workflow filters on monitor tags; a signal-triggered workflow filters on signal attributes. |
| **Fleet Automation** | Faceted inventory search | Agent version, config state, and host tags. Useful for verifying deployment coverage during a migration. |
| **Sensitive Data Scanner** | Log search syntax for scope | The scanning group scope is an ordinary log query; scanning rules are regex-based on top of it. |

---

# 36. Part II decision addendum

``` text
EXTENDED DECISION TREE (Part II surfaces)
=========================================

What are you searching?

├── Front-end user experience
│   └── RUM event search — pick the event type FIRST
│       ├── @type:session   (session-level aggregates)
│       ├── @type:view      (page performance)
│       └── @type:error     (front-end errors)
│
├── Grouped application errors ("which bug?")
│   └── Error Tracking — searches ISSUES, not events
│
├── Builds, pipelines, tests
│   └── CI Visibility — log syntax, but pick the LEVEL
│       └── pipeline / stage / job / command
│
├── Threats and detections
│   └── Signal search
│       ├── status: = SEVERITY here (not log level, not monitor state)
│       └── @workflow.triage.state:(open OR under_review OR archived)
│
├── Reliability targets
│   └── Three surfaces at once
│       ├── SLO list search        (find the SLO object)
│       ├── SLI query              (metric / monitor / time slice)
│       └── error_budget(<id>)     (monitor on top)
│
├── "Who changed this?"
│   └── Audit Trail — @evt.name + @asset.type + @action
│
├── Inventory, joins, aggregate reporting
│   └── DDSQL — SELECT ... FROM dd.<dataset>
│       └── table functions take an event-search string as a parameter
│
└── Tag correctness itself
    └── Tag Policies / metric tag configuration / Audit Trail
```

## Two rules worth repeating

> **`status:` means three different things.** Severity in security signals, log
> level in Logs, monitor state in the Monitor List. Read the surface, then the
> word.

> **Newer surface, weaker guarantee.** Everything in Part II is more likely to
> be entitlement-gated, site-restricted, or in preview than anything in Part I.
> Validate against a live API response before you build automation on it.
