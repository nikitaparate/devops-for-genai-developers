# Networking for Gen AI Apps — AWS, GCP & Azure
### Series: DevOps & Cloud for Gen AI Developers — Part 1

---

Networking is the most overlooked layer in Gen AI applications.

Developers moving into Gen AI focus on model selection, prompt engineering, RAG pipelines, and agent design. Rightfully so. But the moment an app hits production, networking decides three things that matter more than the model — latency, cost, and security.

A poorly configured network means your LLM calls are routing through the public internet when they don't need to. Your NAT Gateway is charging per GB on every Bedrock call. Your inference endpoint is publicly accessible when it should be locked inside a private subnet.

These are not edge cases. They are the default if you don't set things up correctly.

This part covers the core networking concepts every Gen AI developer needs to understand before going to production. Simple analogies first, technical detail after. And for every concept — the equivalent service across AWS, GCP, and Azure. Because the concept is identical. Only the name changes.

---

## 1. Virtual Private Network — Your Private Space on the Cloud

Everyone shares the cloud. Thousands of companies, millions of services — all running on the same infrastructure.

So how does your app stay isolated from everyone else?

That's what a Virtual Private Network does.

| Cloud | Service Name |
|---|---|
| AWS | VPC (Virtual Private Cloud) |
| GCP | VPC (Virtual Private Cloud) |
| Azure | VNet (Virtual Network) |

Think of the cloud as a massive apartment building. Your VPC / VNet is your private apartment. You decide who gets in, which rooms connect to each other, and what goes outside.

Without it? Your services are exposed. With it? You control everything.

Now here is where most Gen AI developers make a mistake. They put everything in a public subnet — meaning everything is reachable from the internet. That works for a demo. Not for production.

The right way:

```
Public Subnet  → API Gateway (this is the only thing users touch)
Private Subnet → LLM Inference Service (never public)
Private Subnet → Vector DB (only your agent can reach it)
Private Subnet → Database (completely locked down)
```

Your LLM service, your vector DB, your data — none of it should be publicly accessible. Only your API Gateway sits at the front door. Everything else is behind it, in private subnets.

Simple rule — **if users don't directly call it, it goes in a private subnet.**

Same rule applies whether you are on AWS, GCP, or Azure.

---

## 2. Private Service Connection — The One That Saves You Money

Okay this one is important. Especially if you are calling managed LLM services — AWS Bedrock, GCP Vertex AI, or Azure OpenAI — from your app.

| Cloud | Service Name |
|---|---|
| AWS | VPC Endpoint |
| GCP | Private Service Connect |
| Azure | Private Endpoint |

Here is what happens without it:

```
Your Service → NAT Gateway → Public Internet → Bedrock / Vertex AI / Azure OpenAI
```

NAT charges per GB of data processed. Sounds small. But think about it — your agent is sending prompts, receiving responses, reading documents. That data adds up very fast.

With a Private Endpoint:

```
Your Service → Private Endpoint → Bedrock / Vertex AI / Azure OpenAI
                                  (stays inside cloud network)
```

No public internet. No NAT cost. Faster. More secure.

**On AWS specifically — two types:**

| Type | Works with | Cost |
|---|---|---|
| Gateway Endpoint | S3, DynamoDB | Free |
| Interface Endpoint | Bedrock, SQS, most AWS services | ~$7/month |

GCP Private Service Connect and Azure Private Endpoint work similarly — one private connection to the managed service, traffic never leaves the cloud network.

If you are calling any managed LLM service — set this up. One config change, real savings at scale.

---

## 3. CDN — Not Just for Static Files

Most developers think CDN is only for serving images, CSS, JS files. I thought the same.

But for Gen AI apps, it does a lot more.

| Cloud | Service Name |
|---|---|
| AWS | CloudFront |
| GCP | Cloud CDN |
| Azure | Azure CDN / Azure Front Door |

A CDN has edge locations across the world. When a user in Mumbai hits your app, the CDN serves them from the nearest edge location — not from your server sitting in some data center thousands of miles away.

So why does this matter for Gen AI?

Think about this. You have an AI assistant. 1000 users ask "what is your refund policy?" Your LLM answers that question 1000 times — paying for tokens each time.

With CDN caching? LLM answers once. CDN serves the cached response to the remaining 999 users. You just saved 999 LLM calls.

Beyond caching:
- **Global low latency** — your app feels fast everywhere, not just near your cloud region
- **DDoS protection** — absorbs traffic spikes before they hit your backend
- **SSL termination** — HTTPS handled at the edge, not your server

For Gen AI apps with high traffic — CDN is not optional. It's essential.

**Azure note:** Azure Front Door is the more powerful option for Gen AI apps — it combines CDN, load balancing, and WAF in one service.

---

## 4. DDoS Protection — Because Your AI App is a Target

You might think — who would attack my AI app?

More people than you think.

DDoS attacks — where thousands of fake requests flood your app at once — are common. And for Gen AI apps, they are particularly dangerous. Why? Because every request to your LLM costs money. An attacker doesn't need to crash your app. They just need to flood it long enough to drain your cloud bill.

| Cloud | Service | Docs |
|---|---|---|
| AWS | Shield | [AWS Shield](https://aws.amazon.com/shield/) |
| GCP | Cloud Armor | [Cloud Armor](https://cloud.google.com/armor) |
| Azure | DDoS Protection | [Azure DDoS](https://azure.microsoft.com/en-us/products/ddos-protection) |

Simple analogy — your AI app is a shop. A DDoS attack is 10,000 fake customers rushing the door, blocking real customers. DDoS protection is the bouncer that spots the fake crowd and blocks them before they reach your shop.

**AWS Shield:**
- Standard — Free, automatic, always on
- Advanced — ~$3000/month, application level protection + 24/7 response team

**GCP Cloud Armor:**
- Pay per policy + per request
- Built into Google's global network — same infrastructure protecting Google Search
- Strong ML-based threat detection

**Azure DDoS Protection:**
- Basic — Free, automatic
- Standard — ~$2500/month, advanced mitigation + cost protection guarantee

DDoS protection alone is not enough for Gen AI. Pair it with:
- **WAF** (Web Application Firewall) — blocks malicious requests, prompt injection attempts
- **API rate limiting** — limit how many requests one user can make
- **LLM Guardrails** — blocks harmful prompts at the model level

Security for Gen AI is layers. Not one tool.

---

## 5. Dedicated Line — Connecting Your Office to Cloud

Not every Gen AI app lives fully in the cloud. Many enterprise apps need to connect to on-premises data centers — legacy databases, internal APIs, sensitive data that cannot move to cloud.

Two options to connect them securely:

| | Dedicated Private Line | VPN |
|---|---|---|
| AWS | Direct Connect | Site-to-Site VPN |
| GCP | Cloud Interconnect | Cloud VPN |
| Azure | ExpressRoute | Azure VPN Gateway |

**VPN:**
Encrypted tunnel over public internet. Quick to set up — hours. Lower cost. Latency depends on your internet quality.

**Dedicated Line (Direct Connect / Interconnect / ExpressRoute):**
A private line from your data center to the cloud. No public internet involved. Consistent low latency. Takes weeks to set up. Higher cost.

Which one for Gen AI?

| Scenario | Use |
|---|---|
| Sending large datasets to fine-tune models | Dedicated Line |
| AI agents querying on-prem databases | VPN is enough |
| Real-time inference needing on-prem data | Dedicated Line |
| Small data, occasional sync | VPN |

If your Gen AI app handles sensitive enterprise data — dedicated line is worth the cost. Your data never touches the public internet.

---

## The Full Picture

Here is what a properly set up Gen AI app networking looks like — same pattern across all three clouds:

```
User Request
    ↓
CDN (CloudFront / Cloud CDN / Azure Front Door)
— edge caching + DDoS protection
    ↓
API Gateway — rate limiting, authentication
    ↓
Load Balancer (public subnet)
    ↓
AI Agent Service (private subnet)
    ↓              ↓                    ↓
Private         Private             Private
Endpoint        Endpoint            Endpoint
(LLM Service)  (Object Storage)    (Database)
```

Everything sensitive is in private subnets. Private Endpoints keep LLM calls off the internet. CDN handles global traffic and caching. DDoS protection blocks attackers.

Your LLM never touches the public internet. That is the goal.

---

## Quick Recap

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| Private Network | VPC | VPC | VNet |
| Private Service Connection | VPC Endpoint | Private Service Connect | Private Endpoint |
| CDN | CloudFront | Cloud CDN | Azure CDN / Front Door |
| DDoS Protection | Shield | Cloud Armor | Azure DDoS Protection |
| Dedicated Line | Direct Connect | Cloud Interconnect | ExpressRoute |
| VPN | Site-to-Site VPN | Cloud VPN | Azure VPN Gateway |

---

## One Thing to Remember

The concepts are identical across all three clouds.

Private network, private endpoints, CDN, DDoS protection, dedicated line — every cloud has them. The names are different. The purpose is the same.
