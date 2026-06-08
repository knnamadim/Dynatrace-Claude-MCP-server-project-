# EasyTrade — Dynatrace Observability Demo
### AI-Assisted Monitoring with MCP Server + Claude + Kubernetes

> **Built by:** Kimberly Nnamadim · Dynatrace SE Capstone · May 2026  
> **Stack:** Dynatrace · AWS EC2 · k3s Kubernetes · EasyTrade · Claude AI · MCP Server

---

## What This Is

This project is a full-stack observability demo built to showcase how Dynatrace — connected to an AI assistant through the **Model Context Protocol (MCP)** — can replace fragmented monitoring toolchains with a single, AI-powered platform.

The demo runs **EasyTrade**, a microservices-based stock trading application on Kubernetes, fully instrumented with Dynatrace. It demonstrates how Davis AI, Grail DQL, and the Dynatrace MCP Server allow engineers and AI assistants to investigate production incidents using natural language — no UI required.

This project was built as a capstone to demonstrate:

- **MCP Server integration** — connecting Claude to live Dynatrace telemetry
- **AI-assisted DevOps** — using natural language to query production observability data
- **Full-stack Kubernetes monitoring** — from infrastructure to business outcomes
- **Enterprise demo skills** — mapping customer pain points to platform capabilities

---

## The Problem This Solves

Most engineering teams dealing with Kubernetes and microservices operate with disconnected toolchains — Grafana, Kibana, CloudWatch, kubectl, PagerDuty — and spend 45–90 minutes manually correlating signals before isolating a root cause.

**EasyTrade's simulated pain points:**

| Pain Point | Description |
|---|---|
| Silent DB write failures | Trades appear accepted but writes fail — detected only after reconciliation |
| Gradual throughput degradation | Platform processes 40–60% fewer requests over time; threshold alerts never fire |
| Third-party payment blind spot | Internal dashboards stay green while customer transactions fail |
| CPU/K8s throttling gap | Infra alerts exist but no visibility into failed trades or revenue impact |
| No business KPI alerting | Trade volume and conversion drops found via customer complaints |
| Broken distributed tracing | Partial OpenTelemetry adoption breaks end-to-end trace visibility |

**What Dynatrace + MCP solves:** a single platform that correlates infrastructure, service behavior, customer experience, and business outcomes — accessible through both a UI and an AI assistant.

---

## Architecture

```
Claude Desktop (AI)
        │
        │  Natural language queries
        ▼
Dynatrace MCP Server ──────────► Dynatrace SaaS (Grail + Davis AI)
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                         OneAgent                ActiveGate
                         (K8s pods)              (routing/proxy)
                              │
                    k3s Kubernetes Cluster (AWS EC2)
                              │
              ┌───────────────┼───────────────────┐
              │               │                   │
       BrokerService    easytrade_offer_service  FlagController
       (C#/.NET)         (Node.js)               (Problem injection)
              │               │
       Engine (Java)    LoginService / AccountService
              │
       MSSQL Database
```

EasyTrade is a polyglot microservices trading app. The frontend is React → Nginx reverse proxy → backend services. The Broker Service handles trade execution. A Feature Flag Service controls problem injection for demo scenarios. A continuous load generator simulates real user traffic.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud | AWS EC2 (t3.2xlarge, Ubuntu 24.04 LTS) |
| Kubernetes | k3s (single-node cluster) |
| Application | EasyTrade (Dynatrace demo app) |
| Observability platform | Dynatrace SaaS |
| Agent deployment | Dynatrace Operator + OneAgent (Helm) |
| Config as Code | Monaco CLI |
| AI assistant | Claude (Anthropic) |
| AI-to-platform bridge | Dynatrace MCP Server |

---

## Key Capabilities Demonstrated

### 1. Davis AI — Automatic Root Cause Analysis
Davis AI auto-baselines every service and detects deviations without manual threshold configuration. It correlates related signals into a single problem with a root cause, affected entities, and timeline — no manual stitching across tools.

**Live example from this environment:**
> Davis detected a response time degradation on `easytrade_offer_service` (447ms vs 184ms baseline, +143%) and linked it to a cascade of JavaScript errors and a failure rate spike on the `:8080` service — all surfaced as a single correlated problem (P-260523).

### 2. MCP Server + Claude — AI-Assisted Investigation
The Dynatrace MCP Server exposes live observability data to Claude through the Model Context Protocol. Engineers (or AI assistants) can query production telemetry using natural language.

**Example prompt used live in this demo:**
```
What problems has Davis detected in my EasyTrade environment in the last 7 days?
What is the root cause of the most recent slowdown?
```

Claude queried the Dynatrace environment through MCP and returned the full Davis problem analysis — same root cause, same correlated entities, same timeline — in seconds, without opening the UI.

### 3. Grail + DQL — Unified Telemetry Queries
Grail stores logs, metrics, traces, events, and business events in a single queryable model. DQL (Dynatrace Query Language) powers investigation across all signal types.

See [`docs/example-queries.md`](docs/example-queries.md) for the full query library used in this demo.

### 4. Real User Monitoring + Session Replay
RUM connects frontend behavior to backend performance. Session Replay allows visual playback of user sessions with error and performance context attached — closing the gap between "backend looks healthy" and actual user experience.

### 5. Synthetic Monitoring
Browser monitors continuously validate key workflows (login journey) from external locations, detecting outages before users report them.

### 6. Workflow Automation
Davis problems trigger automated workflows that route enriched incident context to Slack, email, Jira, or PagerDuty — eliminating the 45–90 minute manual escalation delay.

---

## MCP Server Setup

### Prerequisites
- Claude Desktop installed
- Dynatrace SaaS tenant
- Dynatrace Platform API token (with required permissions)

### Configuration

1. Open your Claude Desktop MCP config file:
```bash
cd ~/Library/Application\ Support/Claude
nano claude_desktop_config.json
```

2. Add the Dynatrace MCP server block:
```json
{
  "mcpServers": {
    "dynatrace": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://<YOUR-TENANT-ID>.apps.dynatrace.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp",
        "--header",
        "Authorization: Bearer <YOUR_PLATFORM_TOKEN>",
        "--transport",
        "sse-first"
      ]
    }
  }
}
```

3. Quit and restart Claude Desktop. If the connection succeeds, Dynatrace tools appear in Claude's tool list.

**Troubleshooting the connection via CLI:**
```bash
npx -y mcp-remote \
  https://<TENANT>.apps.dynatrace.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp \
  --header "Authorization: Bearer <TOKEN>" \
  --transport sse-first
```

Full MCP documentation: [docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp)

---

## Infrastructure Setup

See [`docs/infrastructure-setup.md`](docs/infrastructure-setup.md) for the complete step-by-step guide covering:

- AWS EC2 provisioning (instance type, AMI, security groups, Elastic IP)
- k3s Kubernetes installation and kubeconfig setup
- Helm installation
- Dynatrace Operator deployment via Helm
- OneAgent verification
- EasyTrade deployment via Helm
- NodePort exposure and AWS security group configuration
- Monaco CLI installation and observability configuration deployment
- Feature flag service port-forwarding for problem injection

---

## Problem Injection

EasyTrade uses a Feature Flag Service to trigger real infrastructure failures for demo purposes. Wait 5–20 minutes after triggering before demonstrating — Davis needs time to detect and surface the problem.

**Step 1 — Forward the feature flag service to localhost:**
```bash
kubectl -n easytrade port-forward svc/easytrade-feature-flag-service 8080:8080
```

**Step 2 — Check available flags:**
```bash
curl http://localhost:8080/v1/flags
```

**Step 3 — Trigger the offer service slowdown:**
```bash
curl -X PUT "http://localhost:8080/v1/flags/ergo_aggregator_slowdown" \
  -H "Content-Type: application/json" \
  -d '{"enabled":true}'
```

**Step 4 — Turn it off after the demo:**
```bash
curl -X PUT "http://localhost:8080/v1/flags/ergo_aggregator_slowdown" \
  -H "Content-Type: application/json" \
  -d '{"enabled":false}'
```

---

## Example MCP Prompts

These prompts were used live in this demo to query the Dynatrace environment through Claude:

```
What problems has Davis detected in my EasyTrade environment in the last 7 days?
```
```
What is the root cause of the most recent slowdown?
```
```
Show me all services in the easytrade namespace with elevated error rates.
```
```
What is the response time baseline for easytrade_offer_service?
```
```
Which Kubernetes workloads have had out-of-memory kills in the last 24 hours?
```

See [`docs/example-queries.md`](docs/example-queries.md) for the full DQL query library.

---

## What I Learned

**MCP & AI Integration**
Connecting Claude to live Dynatrace data through MCP is a genuine force multiplier. The same root cause analysis that requires navigating multiple UI screens surfaces in a single natural language prompt. For on-call engineers unfamiliar with an environment, this is the difference between 5 minutes and 45 minutes to first context.

**Kubernetes Observability**
Running a real multi-service application on k3s exposed the actual complexity of Kubernetes monitoring — dynamic service discovery, namespace-scoped telemetry, workload metadata enrichment, and the gap between infrastructure health and business impact. Dynatrace's automatic topology mapping (Smartscape) made dependency relationships visible without manual configuration.

**Demo Engineering as a Skill**
Building a customer-facing SE demo requires thinking about audience at every layer. Executives care about MTTR and revenue impact. SRE teams care about trace correlation and alert noise reduction. Developers care about log-to-trace linkage. Structuring the same platform to speak to all three simultaneously is its own discipline.

---

## References

- [EasyTrade Demo App (Dynatrace)](https://github.com/Dynatrace/easytrade)
- [Capstone Base Repo](https://github.com/kyledharrington/capstone-dynatrace)
- [Dynatrace MCP Server Docs](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp)
- [Monaco Configuration as Code](https://docs.dynatrace.com/docs/deliver/configuration-as-code/monaco)
- [EasyTrade Docker](https://github.com/Dynatrace/easyTravel-Docker)
- [Demo Walkthrough Video](https://www.youtube.com/watch?v=3Gdqn2WI6Go)
