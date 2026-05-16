# Running AI Workloads — Compute, Containers & Kubernetes
### Series: DevOps & Cloud for Gen AI Developers — Part 2

---

Running Gen AI workloads in production is not the same as running a typical web application.

A regular API serves lightweight requests — validate input, query database, return response. A Gen AI app does something fundamentally different. It loads large models, runs inference on GPU or high-memory CPU, manages embedding pipelines, and serves multiple agents simultaneously — each with different compute requirements.

Virtual machines alone cannot handle this cleanly. They are slow to scale, expensive at idle, and hard to replicate consistently across environments. One VM running a slightly different version of your dependencies is enough to cause unpredictable model behaviour.

This is why containers and Kubernetes have become the standard for running Gen AI workloads in production. They give you consistency across environments, fast scaling when traffic spikes, and the ability to run stateless inference services alongside stateful vector databases — each with the right compute attached.

This part covers the compute layer for Gen AI apps. Virtual machines, containers, Kubernetes, and when to use each. AWS, GCP, and Azure — all three covered.

---

## 1. Virtual Machines — The Starting Point

A virtual machine is simply a computer running inside the cloud. You get CPU, RAM, storage — just like your laptop. But in the cloud.

| Cloud | Service | Docs |
|---|---|---|
| AWS | EC2 (Elastic Compute Cloud) | [AWS EC2](https://aws.amazon.com/ec2/) |
| GCP | Compute Engine | [Compute Engine](https://cloud.google.com/compute) |
| Azure | Azure Virtual Machine | [Azure VM](https://azure.microsoft.com/en-us/products/virtual-machines) |

Most Gen AI developers start here. It's familiar. You SSH in, install Python, run your agent. Done.

But VMs have problems at scale:
- **Slow to start** — takes minutes to boot
- **Heavy** — each VM runs a full operating system
- **Hard to replicate** — "works on my VM" is a real problem
- **Expensive at idle** — you pay even when no traffic

For simple use cases or one-off tasks — VMs are fine. For production Gen AI apps with real traffic — you need containers.

**One thing VMs are still great for in Gen AI:** GPU workloads. If you are running your own model inference — you need a GPU VM.

| Cloud | GPU VM Options | Docs |
|---|---|---|
| AWS | EC2 P4, G5 instances | [AWS GPU Instances](https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing) |
| GCP | A100, H100 via Compute Engine | [GCP GPU VMs](https://cloud.google.com/compute/docs/gpus) |
| Azure | NC, ND series VMs | [Azure GPU VMs](https://azure.microsoft.com/en-us/products/virtual-machines/nc-series) |

---

## 2. Block Storage — Persistent Disk for Your VM

Your VM needs somewhere to store data. That's block storage — think of it as a hard drive you attach to your VM.

| Cloud | Service | Docs |
|---|---|---|
| AWS | EBS (Elastic Block Store) | [AWS EBS](https://aws.amazon.com/ebs/) |
| GCP | Persistent Disk | [Persistent Disk](https://cloud.google.com/persistent-disk) |
| Azure | Azure Managed Disk | [Azure Managed Disk](https://azure.microsoft.com/en-us/products/managed-disks) |

Key things to know:
- **Persistent** — data survives even if VM restarts
- **Attachable** — you can detach from one VM and attach to another
- **Scalable** — increase size anytime without stopping the server

For Gen AI specifically — if you are storing model weights or embeddings locally on a VM, block storage is where they live.

But honestly? For most Gen AI apps, store data in object storage (S3 / Cloud Storage / Azure Blob) — not block storage. Cheaper, more scalable, accessible from anywhere.

---

## 3. Containers & Docker — The Game Changer

So what is a container?

Simple analogy — a VM is like renting an entire house. A container is like renting just one room. Same building, but you only use what you need. Faster, cheaper, easier to move around.

A container packages your code + dependencies + config into one portable unit. It runs the same on your laptop, on AWS, on GCP, on Azure. No more "works on my machine."

**For Gen AI apps this matters a lot.** Your agent has specific Python version requirements, specific library versions, specific environment variables. A container locks all of that in. One image, runs anywhere.

### Multi-Stage Docker Builds — Keep Your AI Images Lean

Here is a mistake I made early on. My Docker image was 4GB. Every deployment took forever. Every Kubernetes pod took ages to start.

The fix? Multi-stage builds.

```dockerfile
# Stage 1 — Build (heavy, has all dev tools)
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2 — Runtime (lean, only what's needed to run)
FROM python:3.11-slim
COPY --from=builder /usr/local/lib/python3.11/site-packages ./site-packages
COPY . .
CMD ["python", "agent.py"]
```

Stage 1 installs everything. Stage 2 copies only what's needed to run. Final image = small, fast, secure.

**Docker's rule — last FROM statement = your final image.** Everything before it is discarded. Build tools, dev dependencies, temp files — gone.

For Gen AI apps:

| Stage | What it does |
|---|---|
| Builder | Installs torch, transformers, langchain — heavy |
| Runtime | Copies only installed packages + your code |
| Result | Image goes from 4GB → 800MB |

Use **python:3.11-slim** or **python:3.11-alpine** as your runtime base. Not the full python image.

---

## 4. Container Registry — Where Your Images Live

Once you build a Docker image, you need somewhere to store it so your cloud can pull it.

| Cloud | Service | Docs |
|---|---|---|
| AWS | ECR (Elastic Container Registry) | [AWS ECR](https://aws.amazon.com/ecr/) |
| GCP | Artifact Registry | [Artifact Registry](https://cloud.google.com/artifact-registry) |
| Azure | Azure Container Registry (ACR) | [Azure ACR](https://azure.microsoft.com/en-us/products/container-registry) |

Think of it like GitHub — but for Docker images instead of code.

Your CI/CD pipeline builds the image → pushes to registry → Kubernetes pulls from registry → runs your container.

One tip — **scan your images for vulnerabilities** before pushing to production. All three cloud registries support this built-in. For Gen AI apps, your images have many dependencies (torch, transformers, etc.) — vulnerabilities creep in easily.

---

## 5. Running Containers Without Kubernetes — Serverless Containers

Not every Gen AI app needs Kubernetes. If your app is simple — one or two services — serverless containers are faster to set up and cheaper to run.

| Cloud | Service | Docs |
|---|---|---|
| AWS | ECS Fargate | [AWS Fargate](https://aws.amazon.com/fargate/) |
| GCP | Cloud Run | [Cloud Run](https://cloud.google.com/run) |
| Azure | Azure Container Apps | [Container Apps](https://azure.microsoft.com/en-us/products/container-apps) |

You give it your Docker image. It runs it. Auto-scales on traffic. You pay only when requests come in.

**GCP Cloud Run is particularly good for Gen AI APIs.** It scales to zero when no traffic — you pay nothing. When a request comes in, it spins up in seconds. For low-to-medium traffic AI agents, this is the most cost-efficient option.

When to use serverless containers vs Kubernetes:

| Scenario | Use |
|---|---|
| Simple AI API, low traffic | Serverless containers |
| Multiple microservices, complex routing | Kubernetes |
| Need GPU for inference | Kubernetes (GPU node pools) |
| Want zero infra management | Serverless containers |
| 1000s of pods, auto-scaling control | Kubernetes |

---

## 6. Kubernetes — When Things Get Serious

Kubernetes is a container orchestration platform. Big words. Simple idea.

You tell Kubernetes — "run 10 copies of my AI agent, restart them if they crash, scale up if traffic increases." Kubernetes handles it. You don't manually manage containers.

| Cloud | Managed Kubernetes | Docs |
|---|---|---|
| AWS | EKS (Elastic Kubernetes Service) | [AWS EKS](https://aws.amazon.com/eks/) |
| GCP | GKE (Google Kubernetes Engine) | [GKE](https://cloud.google.com/kubernetes-engine) |
| Azure | AKS (Azure Kubernetes Service) | [AKS](https://azure.microsoft.com/en-us/products/kubernetes-service) |

All three are managed — meaning the cloud handles the Kubernetes control plane. You manage your workloads, not the infrastructure underneath.

### Deployment vs StatefulSet — Know the Difference

This trips up a lot of developers. Two ways to run pods in Kubernetes:

**Deployment — for stateless services**

Every pod is identical. Any pod can die and be replaced. No memory of previous state.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent-api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: agent
        image: my-ecr/ai-agent:latest
```

Use for: AI inference APIs, agent REST APIs, embedding services.

**StatefulSet — for stateful services**

Each pod has a unique identity and its own persistent storage. Pods start and stop in order.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vector-db
spec:
  replicas: 3
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 50Gi
```

Use for: Vector databases, PostgreSQL, Redis, Kafka.

Simple rule — **if it stores data, StatefulSet. If it just processes requests, Deployment.**

### For Gen AI Apps — What Runs Where:

| Component | Type |
|---|---|
| LLM inference API | Deployment |
| AI Agent service | Deployment |
| Vector DB (self-hosted) | StatefulSet |
| Redis (conversation memory) | StatefulSet |
| Kafka (agent message queue) | StatefulSet |
| Embedding service | Deployment |

---

## 7. Horizontal Pod Autoscaler — Scale Automatically

Gen AI apps have unpredictable traffic. A viral moment, a product launch, peak hours — suddenly you need 10x more pods.

HPA (Horizontal Pod Autoscaler) watches your CPU or memory — and automatically scales pods up or down.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: ai-agent-api
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 70
```

When CPU hits 70% → Kubernetes adds more pods automatically. Traffic drops → pods scale back down.

Same concept works on EKS, GKE, and AKS. Same YAML, different cluster.

---

## The Full Picture

Here is how compute and containers fit together for a Gen AI app:

```
Docker Image (multi-stage, slim)
    ↓
Container Registry (ECR / Artifact Registry / ACR)
    ↓
Kubernetes Cluster (EKS / GKE / AKS)
    ↓
┌─────────────────────────────────────┐
│  Deployment (stateless)             │
│  - AI Agent API      (3-50 pods)    │
│  - Inference Service (3-50 pods)    │
│  - Embedding Service (2-20 pods)    │
├─────────────────────────────────────┤
│  StatefulSet (stateful)             │
│  - Vector DB         (3 pods)       │
│  - Redis             (3 pods)       │
│  - Kafka             (3 pods)       │
└─────────────────────────────────────┘
    ↓
HPA scales pods automatically on traffic
```

---

## Quick Recap

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| Virtual Machine | EC2 | Compute Engine | Azure VM |
| Block Storage | EBS | Persistent Disk | Managed Disk |
| Container Registry | ECR | Artifact Registry | ACR |
| Serverless Containers | ECS Fargate | Cloud Run | Container Apps |
| Managed Kubernetes | EKS | GKE | AKS |

---

## One Thing to Remember

Kubernetes YAML files are the same across EKS, GKE, and AKS.

You write a Deployment or StatefulSet once — it runs on any cloud. That's the beauty of Kubernetes. Learn it once, use it everywhere.

The only difference is how you set up the cluster itself. After that — same YAML, same commands, same concepts.
