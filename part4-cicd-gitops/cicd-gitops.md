# Shipping AI Fast — CI/CD, GitOps, ArgoCD & Terraform
### Series: DevOps & Cloud for Gen AI Developers — Part 4

---

Manual deployments are the single biggest operational risk in Gen AI applications.

Not because developers are careless. But because Gen AI apps have more moving parts than a typical web application. A model version. A prompt version. Environment variables pointing to the right LLM endpoint. Guardrail configurations. Vector DB connection strings. Token limits. Each one a potential point of failure when deployed by hand.

Miss one — and your agent calls the wrong model in production. Or ignores your guardrails entirely. Or connects to the staging vector DB instead of production. These are not hypothetical failures. They happen regularly in teams that have not automated their deployment process.

CI/CD and GitOps solve this. Not by making deployments faster — but by making them repeatable, auditable, and reversible. Every change goes through Git. Every deployment is automated. Every rollback is one command.

*This part covers how to set that up for Gen AI apps specifically — including the parts that are different from deploying a regular web application. AWS, GCP, and Azure — all three covered.*

---

## First — What is CI/CD?

Two things combined.

**CI — Continuous Integration:**
Every time you push code, it automatically runs tests, builds your Docker image, checks for issues. No manual steps.

**CD — Continuous Deployment:**
After CI passes, it automatically deploys your new version to the environment. No manual SSH. No manual restarts.

Together — you push code to Git, everything else happens automatically. Your app is live in minutes.

| Cloud | Native CI/CD |
|---|---|
| AWS | CodePipeline + CodeBuild |
| GCP | Cloud Build |
| Azure | Azure DevOps |
| All clouds | GitHub Actions (most popular, works everywhere) |

Honest opinion — most teams use **GitHub Actions** regardless of which cloud they are on. It integrates with AWS, GCP, and Azure equally well. Learn one, use everywhere.

---

## 2. A Basic CI/CD Pipeline for Gen AI Apps

Here is what a pipeline looks like for a Gen AI app. Same structure works on GitHub Actions, Cloud Build, and Azure DevOps.

```
Developer pushes code to Git
    ↓
CI kicks off automatically
    ↓
┌─────────────────────────────────┐
│  Run unit tests                 │
│  (mock LLM calls — don't use    │
│   real API in CI, costs money)  │
├─────────────────────────────────┤
│  Run linting / code quality     │
├─────────────────────────────────┤
│  Build Docker image             │
│  (multi-stage, slim image)      │
├─────────────────────────────────┤
│  Push image to registry         │
│  (ECR / Artifact Registry / ACR)│
└─────────────────────────────────┘
    ↓
CD kicks off
    ↓
Deploy to staging → run integration tests → deploy to production
```

Simple GitHub Actions example:

```yaml
name: Deploy AI Agent

on:
  push:
    branches: [main]
    paths:
      - 'src/**'           # only trigger if source code changed
      - 'Dockerfile'       # or if Dockerfile changed
      # ignore: README, docs, notebooks

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

      - name: Install and test
        run: |
          pip install -r requirements.txt
          pytest tests/unit/    # unit tests only — no real LLM calls

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t my-ai-agent:${{ github.sha }} .

      - name: Push to registry
        run: docker push my-registry/my-ai-agent:${{ github.sha }}

      - name: Update Kubernetes deployment
        run: |
          kubectl set image deployment/ai-agent \
            agent=my-registry/my-ai-agent:${{ github.sha }}
```

---

## 3. CI/CD Optimizations for Gen AI Apps

Gen AI apps have specific things that slow down pipelines and increase costs if you are not careful.

**Never call real LLMs in CI.**

This is the most common mistake. Developers run integration tests in CI that call actual Bedrock, Vertex AI, or Azure OpenAI. Every pipeline run costs real tokens. Costs add up fast.

Mock your LLM calls in tests:

```python
# Don't do this in CI
response = bedrock_client.invoke_model(...)

# Do this instead
@patch("boto3.client")
def test_agent_response(mock_client):
    mock_client.return_value.invoke_model.return_value = {
        "body": MockResponse("test output")
    }
    result = my_agent.process("test input")
    assert result is not None
```

Real LLM integration tests? Run them only on merge to main. Not on every PR.

**Cache model weights and dependencies:**

```yaml
- name: Cache Hugging Face models
  uses: actions/cache@v3
  with:
    path: ~/.cache/huggingface
    key: models-${{ hashFiles('requirements.txt') }}
```

Downloading a 7B model on every pipeline run kills your build time. Cache it.

**Use Alpine / slim images:**

```dockerfile
# Bad — full image, 1.1GB
FROM python:3.11

# Good — slim image, 180MB
FROM python:3.11-slim
```

Smaller image = faster push to registry = faster pull in Kubernetes = faster deployment.

**Run jobs in parallel:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest    # runs in parallel
  lint:
    runs-on: ubuntu-latest    # runs in parallel
  security-scan:
    runs-on: ubuntu-latest    # runs in parallel
  build:
    needs: [test, lint, security-scan]  # waits for all three
```

Test, lint, and security scan run simultaneously. Build only starts when all three pass. Cuts pipeline time significantly.

**Only trigger on what changed:**

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'requirements.txt'
```

If someone updates the README or adds a notebook — no pipeline runs. Save compute minutes.

---

## 4. GitOps — Git as the Source of Truth

CI/CD automates the build and deploy. GitOps takes it one step further.

**GitOps principle: whatever is in your Git repo = exactly what runs in production.**

No manual kubectl commands. No config changes directly on the server. No "I'll just quickly update this environment variable in the console." Everything goes through Git.

Why does this matter?

- **Auditability** — every change is a Git commit. You know who changed what and when.
- **Rollback** — something broke? Git revert = instant rollback to previous state.
- **Consistency** — staging and production are identical because they both come from the same Git repo.
- **Drift detection** — if someone manually changes something in production, GitOps detects it and alerts you.

Simple analogy — your Git repo is the **blueprint of your house**. GitOps ensures the actual house always matches the blueprint. Someone moves a wall without updating the blueprint? GitOps flags it immediately.

---

## 5. ArgoCD — GitOps for Kubernetes

ArgoCD is the tool that implements GitOps for Kubernetes. It watches your Git repo and automatically keeps your Kubernetes cluster in sync with whatever is in Git.

Works the same on EKS, GKE, and AKS. Cloud agnostic.

```
Developer updates Kubernetes YAML in Git
    ↓
ArgoCD detects the change (watches repo continuously)
    ↓
ArgoCD applies the change to Kubernetes cluster
    ↓
Cluster now matches Git exactly
```

**Without ArgoCD:**
```
Developer → manually runs kubectl apply → hope it worked → forget what was deployed
```

**With ArgoCD:**
```
Developer → pushes YAML to Git → ArgoCD handles everything → cluster always matches Git
```

ArgoCD gives you a visual dashboard showing every app, every pod, every deployment status. Green = matches Git. Red = something drifted. You can sync, rollback, and inspect — all from the UI.

Basic ArgoCD application config:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ai-agent
spec:
  source:
    repoURL: https://github.com/myorg/ai-agent-config
    path: kubernetes/
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # remove resources deleted from Git
      selfHeal: true    # auto fix if someone manually changes cluster
```

`selfHeal: true` is the key setting. If someone manually runs kubectl and changes something — ArgoCD detects it and reverts it back to what Git says. No more manual changes sneaking into production.

---

## 6. Terraform — Your Infrastructure as Code

ArgoCD manages what runs inside Kubernetes. But who created the Kubernetes cluster itself? Who created the VPC, the subnets, the ECR registry, the RDS database?

That's Terraform.

| Cloud | Native IaC | Terraform |
|---|---|---|
| AWS | CloudFormation | ✅ Works |
| GCP | Deployment Manager | ✅ Works |
| Azure | ARM Templates / Bicep | ✅ Works |

Terraform is cloud agnostic. One tool, all three clouds. That's why most teams pick Terraform over the native alternatives.

**Without Terraform:**
```
Login to AWS Console
→ Click create VPC
→ Click create EKS cluster
→ Click create RDS
→ Do it again for staging environment
→ Hope staging matches production
```

**With Terraform:**
```
Write config once
→ terraform apply
→ Everything created automatically
→ Run same config for staging = identical environment
```

Basic Terraform for Gen AI infrastructure:

```hcl
# Create EKS cluster for Gen AI workloads
resource "aws_eks_cluster" "ai_cluster" {
  name     = "ai-agent-cluster"
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}

# Create ECR registry for AI agent images
resource "aws_ecr_repository" "ai_agent" {
  name                 = "ai-agent"
  image_tag_mutability = "IMMUTABLE"  # prevents overwriting images

  image_scanning_configuration {
    scan_on_push = true   # auto scan for vulnerabilities
  }
}

# Create S3 bucket for documents / RAG data
resource "aws_s3_bucket" "ai_documents" {
  bucket = "my-ai-agent-documents"
}
```

Run `terraform plan` first — it shows you exactly what will be created, changed, or destroyed. Before touching anything. That's the safety net.

**Terraform on GCP:**
```hcl
resource "google_container_cluster" "ai_cluster" {
  name     = "ai-agent-cluster"
  location = "us-central1"
}
```

**Terraform on Azure:**
```hcl
resource "azurerm_kubernetes_cluster" "ai_cluster" {
  name                = "ai-agent-cluster"
  location            = "East US"
  resource_group_name = azurerm_resource_group.rg.name
}
```

Same structure. Different provider. Same Terraform knowledge.

---

## 7. Deployment Strategies — Ship Safely

When you deploy a new version of your Gen AI app, how do you do it without breaking production?

Five strategies. Each with different risk vs speed tradeoffs.

**Recreate** — kill all old pods, deploy new ones. Causes downtime. Never use in production.

**Rolling** — replace old pods one by one. No downtime. Default in Kubernetes.

**Blue/Green** — run old version (blue) and new version (green) side by side. Switch traffic instantly. Easy rollback.

**Canary** — send 5% of traffic to new version. Watch metrics. Gradually increase if stable.

**A/B Testing** — split traffic to test different versions on different users. Good for testing prompt changes.

**For Gen AI apps specifically:**

| Scenario | Strategy |
|---|---|
| Regular code update | Rolling |
| Major model change | Blue/Green |
| New prompt version | Canary — test on 5% first |
| Testing two different LLMs | A/B Testing |
| Emergency hotfix | Recreate (accept brief downtime) |

Canary deployments are particularly valuable for Gen AI apps. A bad prompt or model change can silently degrade response quality without throwing errors. Canary lets you catch it on 5% of users before it hits everyone.

---

## The Full CI/CD + GitOps Picture

```
Developer pushes code to Git
    ↓
GitHub Actions / Cloud Build / Azure DevOps
    ↓
┌─────────────────────────────────────┐
│  Parallel jobs:                     │
│  - Unit tests (mocked LLM calls)    │
│  - Lint + code quality              │
│  - Security scan                    │
└─────────────────────────────────────┘
    ↓ all pass
Build Docker image (multi-stage, slim)
    ↓
Push to ECR / Artifact Registry / ACR
    ↓
Update Kubernetes YAML in Git repo
    ↓
ArgoCD detects Git change
    ↓
ArgoCD deploys to Kubernetes (EKS / GKE / AKS)
    ↓
Canary → Rolling → Full deployment
    ↓
Terraform manages underlying infrastructure
```

---

## Quick Recap

| Concept | AWS | GCP | Azure | Cloud Agnostic |
|---|---|---|---|---|
| CI/CD | CodePipeline | Cloud Build | Azure DevOps | GitHub Actions |
| Container Registry | ECR | Artifact Registry | ACR | — |
| GitOps | ArgoCD | ArgoCD | ArgoCD | ArgoCD |
| Infrastructure as Code | CloudFormation | Deployment Manager | ARM / Bicep | Terraform |
| Managed Kubernetes | EKS | GKE | AKS | — |

---

## One Thing to Remember

ArgoCD and Terraform are cloud agnostic.

Learn them once — use on AWS, GCP, or Azure without relearning. That's the real advantage. Most of your CI/CD and GitOps knowledge transfers across clouds. Only the specific service names change.

And never — ever — deploy Gen AI apps manually. One wrong environment variable in production and your agent calls the wrong model, uses the wrong API key, or ignores your guardrails entirely. Automate everything. Let Git be your source of truth.

