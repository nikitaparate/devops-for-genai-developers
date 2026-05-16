# Observability for Gen AI Apps — Metrics, Logs & Traces
### Series: DevOps & Cloud for Gen AI Developers — Part 3

---

Traditional monitoring was built for traditional applications.

Is the server up? Is CPU below 80%? Is memory within limits? For a REST API or a database, these questions are sufficient. If the infrastructure is healthy, the application is healthy.

Gen AI applications break this assumption entirely.

Your Kubernetes pods can show green across every infrastructure metric — normal CPU, normal memory, no error rates — while your application is silently failing. The LLM is taking 40 seconds to respond. Your RAG retrieval is returning irrelevant documents. A single user is consuming 60% of your token budget. Your agent is looping on a subtask and burning tokens on every iteration.

None of this shows up in CloudWatch, Cloud Monitoring, or Azure Monitor out of the box. Because these tools measure infrastructure. Gen AI applications need a completely different observability layer — one that understands models, tokens, prompts, agent steps, and LLM costs.

This part covers what that layer looks like. What to measure, which tools to use, and how to build visibility into the parts of your Gen AI app that traditional monitoring cannot see. AWS, GCP, and Azure — all three covered.

---

## First — What is Observability?

Three pillars. Remember these.

**Metrics** — numbers over time. CPU 70%, latency 200ms, token count 1500.

**Logs** — what happened and when. "Agent called Bedrock at 14:32:01, got response in 8s."

**Traces** — the full journey of one request across all services. User request → Agent → Vector DB → LLM → Response.

Most developers only set up metrics. That's like checking your car's fuel gauge and ignoring the engine warning light and the GPS at the same time.

You need all three. Especially for Gen AI.

---

## 1. Cloud Native Monitoring — What It Tracks and What It Doesn't

Every cloud gives you a built-in monitoring service. Free, automatic, no setup.

| Cloud | Service | Docs |
|---|---|---|
| AWS | CloudWatch | [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) |
| GCP | Cloud Monitoring | [Cloud Monitoring](https://cloud.google.com/monitoring) |
| Azure | Azure Monitor | [Azure Monitor](https://azure.microsoft.com/en-us/products/monitor) |

**What they track automatically:**

| Resource | Metrics |
|---|---|
| VM / Compute | CPU, Memory, Network In/Out, Disk I/O |
| Kubernetes | Pod restarts, node CPU, memory per pod |
| Serverless Functions | Invocation count, duration, errors, throttles |
| API Gateway | Request count, latency, 4XX/5XX errors |
| Database | Connections, read/write latency, storage |
| Object Storage | Request count, bytes transferred |

Good for knowing — is my infrastructure alive and healthy?

**What they cannot track:**

This is the important part. Native monitoring is blind to:

- `token_usage_per_user` — which user is burning your LLM budget
- `llm_response_latency_ms` — how long the model itself took vs your code
- `prompt_size_over_time` — are your prompts growing and costing more
- `agent_step_duration_ms` — which step in your agent pipeline is the bottleneck
- `vector_search_latency_ms` — how long your RAG retrieval is taking
- `hallucination_rate` — how often your LLM gives wrong answers
- `cost_per_conversation` — what each user session actually costs you
- `embedding_generation_time_ms` — time spent converting text to vectors

These are Gen AI specific metrics. No cloud gives them to you automatically. You have to build them yourself — and push them to your monitoring tool.

---

## 2. Distributed Tracing — Follow One Request Across Everything

Here is the hardest problem in Gen AI observability.

A user sends a message. Your agent processes it. Internally — it calls your vector DB, calls the LLM, formats the response, logs the conversation. Five or six steps. Each step takes time.

Your app responds in 12 seconds. Which step caused it?

Without distributed tracing — you have no idea. You are guessing.

With distributed tracing — you see this:

```
User Request — total: 12s
  ├── Auth check — 50ms
  ├── Vector DB search — 800ms
  ├── Prompt construction — 100ms
  ├── LLM call (Bedrock / Vertex / Azure OpenAI) — 10.2s  ← here
  └── Response formatting — 50ms
```

The LLM call took 10.2 seconds. Now you know. Maybe your prompt is too large. Maybe you need to switch models. Maybe you need streaming.

| Cloud | Native Tracing | Docs |
|---|---|---|
| AWS | X-Ray | [AWS X-Ray](https://aws.amazon.com/xray/) |
| GCP | Cloud Trace | [Cloud Trace](https://cloud.google.com/trace) |
| Azure | Application Insights | [Application Insights](https://azure.microsoft.com/en-us/products/monitor) |

**Limitation of native tracing:**

X-Ray, Cloud Trace, and Application Insights trace within their own cloud services well. But if your agent calls an external API — OpenAI, Anthropic directly, Pinecone, a third party — native tracing goes blind at that boundary.

That's where Datadog and Grafana come in.

---

## 3. Datadog — When You Need Everything in One Place

Datadog is a third party observability platform. Works across AWS, GCP, and Azure — all in one dashboard.

**What Datadog gives you that native monitoring doesn't:**

- **APM (Application Performance Monitoring)** — traces inside your code, not just between services. See which function, which line is slow.
- **Log correlation** — one click from a metric spike → to the logs → to the trace. All connected.
- **Custom metrics** — push your own Gen AI metrics. Token usage, cost per user, agent step duration.
- **Anomaly detection** — ML-based alerting. Alerts you before things break, not after.
- **Real User Monitoring** — actual experience of your users in the browser or app.
- **Cross-cloud** — one dashboard for AWS + GCP + Azure if you use multiple clouds.

**For Gen AI specifically — Datadog LLM Observability:**

Datadog now has built-in LLM monitoring. It tracks:
- `llm.request.duration` — how long each LLM call takes
- `llm.tokens.input` / `llm.tokens.output` — token usage per call
- `llm.cost` — estimated cost per call
- `llm.errors` — failed LLM calls

You instrument it once in your code:

```python
from ddtrace.llmobs import LLMObs

LLMObs.enable(ml_app="my-ai-agent")

# Your LLM call is now automatically traced
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": prompt}]
)
```

That's it. Token usage, latency, cost — all flowing into Datadog automatically.

---

## 4. Grafana — The Open Source Alternative

Grafana is open source. Free. You host it yourself or use Grafana Cloud.

The real power of Grafana is — it connects to everything.

| Data Source | What it gives you |
|---|---|
| Prometheus | Kubernetes metrics, custom app metrics |
| Loki | Log aggregation from all pods |
| Tempo | Distributed traces |
| CloudWatch | AWS metrics |
| Cloud Monitoring | GCP metrics |
| Azure Monitor | Azure metrics |

One Grafana dashboard can show you AWS metrics, GCP metrics, your Kubernetes pod logs, and your custom Gen AI metrics — all in one place.

**Grafana + Loki for logs:**

If you have 1000 pods running your Gen AI app — you cannot check logs one pod at a time. Loki collects logs from all pods centrally. Grafana lets you search across all of them in one query.

```
{app="ai-agent"} |= "error" | json | latency > 5000
```

That query finds every error log across all your AI agent pods where latency exceeded 5 seconds. Across your entire cluster. In seconds.

**Grafana vs Datadog — which to pick:**

| | Datadog | Grafana |
|---|---|---|
| Cost | Expensive | Free / low cost |
| Setup | Easy | More work |
| LLM monitoring | Built-in | Custom setup needed |
| Multi-cloud | Yes | Yes |
| Best for | Teams wanting all-in-one | Teams comfortable with open source |

---

## 5. Purpose-Built LLM Observability Tools

Native monitoring doesn't know what an LLM is. Datadog and Grafana are general purpose. But there is a new category of tools built specifically for LLM apps.

| Tool | What it does |
|---|---|
| **Langfuse** | Open source. Traces every LLM call, prompt version, token cost. |
| **Helicone** | Proxy-based. Zero code change — just route calls through it. |
| **LangSmith** | By LangChain team. Best for LangChain / LangGraph agents. |
| **Arize Phoenix** | Open source. Evaluates LLM output quality, hallucinations. |

These tools track things no general monitoring tool tracks:

- **Prompt versions** — did changing the prompt improve or hurt quality?
- **Hallucination detection** — is the LLM making things up?
- **Output quality scoring** — automated evaluation of LLM responses
- **Cost per session** — exact dollar cost of each user conversation
- **Token breakdown** — system prompt vs user message vs context

For production Gen AI apps — I'd recommend using one of these alongside your cloud monitoring. They are built for exactly this problem.

---

## 6. What to Monitor for Gen AI Apps — The Full List

Here is exactly what you should be tracking. Split into three layers.

**Infrastructure layer** (cloud native monitoring handles this):
- `cpu_utilization_%`
- `memory_utilization_%`
- `pod_restart_count`
- `api_gateway_latency_ms`
- `error_rate_%`

**Application layer** (Datadog / Grafana / custom metrics):
- `agent_pipeline_duration_ms` — end to end
- `vector_search_latency_ms`
- `embedding_generation_time_ms`
- `llm_call_duration_ms`
- `queue_depth` — messages waiting to be processed

**Gen AI layer** (LLM observability tools):
- `token_usage_per_request`
- `token_usage_per_user`
- `cost_per_conversation`
- `prompt_size_over_time`
- `llm_error_rate`
- `hallucination_rate`
- `response_quality_score`
- `cache_hit_rate` — how often you avoided an LLM call

---

## 7. Alerting — Know Before Your Users Do

Monitoring without alerting is just a pretty dashboard nobody looks at.

Set alerts for these specifically for Gen AI apps:

| Alert | Threshold | Why |
|---|---|---|
| LLM latency | > 10s | Users will abandon |
| Token usage spike | > 2x normal | Possible prompt injection attack |
| Error rate | > 1% | Something is broken |
| Cost per hour | > budget limit | Runaway spending |
| Vector DB latency | > 500ms | RAG retrieval is bottleneck |
| Pod restart count | > 3 in 5 mins | Service is crashing |

All three cloud monitoring tools support alerting. For more advanced alerting — PagerDuty and Opsgenie integrate with Datadog, Grafana, CloudWatch, and Cloud Monitoring.

---

## The Full Observability Stack for Gen AI

```
Your Gen AI App
    ↓
Emit logs + metrics + traces
    ↓
┌─────────────────────────────────────────┐
│  Infrastructure Layer                   │
│  CloudWatch / Cloud Monitoring /        │
│  Azure Monitor                          │
│  — CPU, memory, pods, API latency       │
├─────────────────────────────────────────┤
│  Application Layer                      │
│  Datadog / Grafana + Prometheus + Loki  │
│  — Traces, custom metrics, log search   │
├─────────────────────────────────────────┤
│  LLM Layer                              │
│  Langfuse / Helicone / LangSmith        │
│  — Token cost, prompt versions,         │
│    hallucinations, output quality       │
└─────────────────────────────────────────┘
    ↓
Alerts → PagerDuty / Opsgenie / Slack
```

---

## Quick Recap

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| Native Monitoring | CloudWatch | Cloud Monitoring | Azure Monitor |
| Distributed Tracing | X-Ray | Cloud Trace | Application Insights |
| Log Management | CloudWatch Logs | Cloud Logging | Log Analytics |
| Third Party (all clouds) | Datadog | Datadog | Datadog |
| Open Source (all clouds) | Grafana + Loki | Grafana + Loki | Grafana + Loki |
| LLM Specific (all clouds) | Langfuse / Helicone / LangSmith | Same | Same |

---

## One Thing to Remember

Native cloud monitoring tells you your infrastructure is alive.

Datadog and Grafana tell you your application is healthy.

LLM observability tools tell you your AI is actually working correctly.

You need all three layers. Most teams set up the first, skip the second, and completely ignore the third. That's why they find out about problems from users — not from their dashboards.

