# Example Queries & MCP Prompts

DQL queries and Claude MCP prompts used in the EasyTrade Dynatrace demo.

---

## MCP Prompts (Natural Language via Claude)

These prompts query live Dynatrace data through the MCP Server connected to Claude Desktop.

### Problem Investigation
```
What problems has Davis detected in my EasyTrade environment in the last 7 days?
```
```
What is the root cause of the most recent slowdown?
```
```
Are there any open problems in my environment right now?
```
```
Which services have had the most problems in the last 7 days?
```

### Service Health
```
Show me all services in the easytrade namespace with elevated error rates.
```
```
What is the current response time for easytrade_offer_service compared to its baseline?
```
```
Which Kubernetes workloads have had out-of-memory kills in the last 24 hours?
```
```
Is the BrokerService healthy right now?
```

### Root Cause & Impact
```
What is causing the slowdown in easytrade_offer_service?
```
```
How many users were impacted during the most recent error spike?
```
```
What downstream services were affected by the last Davis problem?
```

---

## DQL Queries (Dynatrace Notebooks / Dashboards)

### Pain A — Silent Errors by Pod (Last 2 hours)
Identifies which pods are generating hidden errors that don't surface in dashboards.

```dql
fetch logs
| filter k8s.namespace.name == "easytrade"
| filter matchesPhrase(content, "error")
    or matchesPhrase(content, "exception")
    or matchesPhrase(content, "failed")
| summarize errorCount = count(), by: {k8s.pod.name}
| sort errorCount desc
| limit 10
```

**Business value:** Finds trade failures and write errors before customers notice them.

---

### Pain B — Service Response Time Slow Bleed (24 hours)
Tracks gradual performance degradation that threshold-based alerts miss entirely.

```dql
timeseries avg(dt.service.request.response_time),
  by: {dt.entity.service},
  filter: {k8s.namespace.name == "easytrade"}
| limit 500
```

**Business value:** Catches the 40–60% throughput degradation scenario before it becomes a P1 incident.

---

### Pain C — Third-Party Dependency Blind Spot
Surfaces external dependencies that aren't formally instrumented.

```dql
fetch dt.entity.service
| filter matchesPhrase(entity.name, "Requests to unmonitored hosts")
| fields entity.name, entity.type, tags
```

**Business value:** Reveals payment processor or external API failures even when internal dashboards show green. Davis auto-detects these dependencies even without explicit instrumentation.

---

### Pain D — Business KPI Funnel (Trade Activity)
Tracks buy/sell activity in 5-minute windows to connect technical performance to business outcomes.

```dql
fetch bizevents
| filter event.type == "com.easytrade.buy.start"
    or event.type == "com.easytrade.buy.finish"
    or event.type == "com.easytrade.sell.start"
    or event.type == "com.easytrade.sell.finish"
| summarize count = count(), by:{event.type, interval = bin(start_time, 5m)}
```

**Business value:** Connects a service degradation directly to a drop in trade completions — the conversation executives care about.

---

### Service Response Time Trends
```dql
timeseries avg(dt.service.request.response_time),
  by: {dt.entity.service},
  filter: {k8s.namespace.name == "easytrade"}
| limit 500
```

---

### Log Filtering — EasyTrade Errors
```dql
fetch logs
| filter k8s.namespace.name == "easytrade"
| filter loglevel in ["ERROR", "WARN"]
```

---

### Log Filtering — BrokerService Errors
```dql
fetch logs
| filter k8s.namespace.name == "easytrade"
| filter service.name == "BrokerService"
| filter loglevel in ["ERROR", "WARN"]
```

---

### Combined Namespace + Service Log Filter
```dql
fetch logs
| filter k8s.namespace.name == "easytrade"
| filter service.name == "BrokerService"
| filter loglevel in ["ERROR", "WARN"]
```

---

### Davis Problems (Last 7 Days)
```dql
fetch dt.davis.problems, from:now() - 7d
| fields event.id, display_id, event.status, event.start, event.end,
         event.category, event.description, affected_entity_ids
| sort event.start desc
| limit 50
```

---

## Demo Screens to Bookmark

These are the screens to have open or pre-loaded before a live demo:

| Screen | Purpose |
|---|---|
| Davis Problems App (7d) | Show the full incident history |
| Specific problem: `easytrade_offer_service` P-260514 | Best slowdown example — clean root cause |
| Services Explorer | Show service health and properties/metadata |
| Distributed Tracing Explorer | Show full request waterfall |
| Logs App | Filter by `k8s.namespace.name = easytrade` |
| EasyTrade Web RUM App | Show Apdex, user journey, frontend errors |
| Session Replay | Pre-select a session with an error |
| Synthetic Monitor: EasyTrade Login Journey | Show browser monitor health |
| Workflow: EasyTrade Problem Notification | Show problem trigger + email action |
| Notebooks | Pre-run all 4 DQL queries |
| Dashboard | Show open problems + service performance tiles |
| Kubernetes namespace view: `easytrade` | Show workload topology |

---

## Feature Flags Reference

All flags available in the EasyTrade Feature Flag Service:

| Flag | Effect |
|---|---|
| `ergo_aggregator_slowdown` | Triggers response time degradation on `easytrade_offer_service` — maps to Pain B |
| Other flags | Run `curl http://localhost:8080/v1/flags` to see all available flags |

**Port-forward the service before using:**
```bash
kubectl -n easytrade port-forward svc/easytrade-feature-flag-service 8080:8080
```
