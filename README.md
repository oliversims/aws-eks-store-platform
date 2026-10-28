<h1 align="center">Store on Amazon EKS</h1>

Private GitOps on Amazon EKS for an eight-service store. Terraform owns the platform. GitHub Actions builds images into private ECR. Helm values in Git are the deploy contract. Argo CD reconciles the cluster. A published GitHub Release is the only path that changes what the cluster runs.

AWS compute was destroyed after the working run so idle EKS, NAT, and load balancer cost does not keep accruing. Hostnames below do not resolve today. Proof is the architecture diagrams and screenshots in this README.

> This repository presents the project. The source code is private and available on request.

[![AWS](https://img.shields.io/badge/AWS-us--east--1-232F3E?logo=amazonwebservices&logoColor=white)](https://aws.amazon.com/)
[![EKS](https://img.shields.io/badge/Amazon%20EKS-1.34-FF9900?logo=amazoneks&logoColor=white)](https://aws.amazon.com/eks/)
[![Terraform](https://img.shields.io/badge/Terraform-1.x-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Helm](https://img.shields.io/badge/Helm-chart-0F1689?logo=helm&logoColor=white)](https://helm.sh/)
[![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-OIDC-2088FF?logo=githubactions&logoColor=white)](https://github.com/features/actions)

| | |
| --- | --- |
| **Region** | `us-east-1` |
| **Environment** | `store-dev` · cluster `store-dev-store` |
| **Network** | VPC `10.0.0.0/16` · three Availability Zones · one NAT Gateway |
| **Workloads** | Eight services in namespace `store` · HPA 3–6 at 70% CPU |
| **Delivery** | GitHub Actions OIDC → private ECR · Release opens a Helm-values PR · Argo CD syncs |
| **Data** | DynamoDB catalog · Amazon MQ orders · DocumentDB completed orders |
| **Edge** | Store ALB (WAF, ACM) · ops ALB for Grafana and Argo CD |
| **IaC** | One Terraform stack per concern · S3 remote state |
| **Status** | Torn down after the demo · recreate and destroy runbooks in the private repositories |

---

## Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Design choices](#design-choices)
4. [Source layout (private repositories)](#source-layout-private-repositories)
5. [CI/CD](#cicd)
6. [Infrastructure](#infrastructure)
7. [Application (working run)](#application-working-run)
8. [Security snapshot](#security-snapshot)
9. [Observability](#observability)
10. [Operations](#operations)

---

## Overview

The store is eight services: Vue storefront and admin, Node, Rust, Go, Python, plus two in-cluster simulators. This repository is the public walkthrough of the platform around them: isolated network, gated image pipeline, GitOps deploy, and production-shaped controls on a single environment.

Infrastructure is Terraform with encrypted remote state in S3. Each stack has one job. Later stacks read IDs from earlier state instead of copying them by hand.

What this project is meant to show a hiring manager:

| Goal | How it is met |
| --- | --- |
| Repeatable infrastructure | Ordered Terraform stacks, remote state, apply/destroy scripts |
| Gated delivery | Format, tests, Gitleaks, Semgrep, and Trivy on GitHub Actions before ECR push |
| GitOps | A GitHub Release edits Helm tags in Git. A person merges. Argo CD reconciles. Actions never receive a kubeconfig. |
| Least privilege | Private subnets, Pod Identity, Secrets Manager, default-deny NetworkPolicy, PSA `restricted` |
| Operable teardown | Destroy is the reverse of apply; the hosted zone and ACM cert stay |

---

## Architecture

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Platform architecture</b></p>
      <img src="docs/architecture/store-architecture.png" alt="Store platform on AWS — CI, VPC, EKS, data stores, and observability" width="100%">
      <br><sub><b>Figure 1.</b> Region us-east-1 · VPC 10.0.0.0/16 · three Availability Zones.</sub>
    </td>
  </tr>
</table>

| Plane | What runs | Network |
| --- | --- | --- |
| Edge | Store ALB (WAF) · ops ALB | Public subnets · HTTPS `:443` · HTTP `:80` redirects to 443 |
| Control | EKS 1.34 API · Argo CD · Grafana | Private nodes · ops ALB for the UIs |
| App | Namespace `store` · eight workloads | Private subnets · ClusterIP behind the store Ingress |
| Data | DynamoDB · Amazon MQ · DocumentDB | Private · no public addresses |
| Build | GitHub Actions · private ECR | OIDC role · no AWS access key in GitHub |

Public subnets hold the internet gateway, NAT gateway, and load balancers. Private subnets hold the nodes and the data stores. The managed node group is the baseline. Karpenter adds nodes when the baseline is full.

### Request path

```
User  →  Route 53 (store.simsoliver.com / store-admin.simsoliver.com)
      →  Store ALB :443  (TLS terminates here, WAF)
      →  store-front :8080  /  store-admin :8081
      →  product-service :3002 · order-service :3000 · makeline-service :3001 · ai-service :5001
      →  DynamoDB · Amazon MQ :5671 · DocumentDB :27017 · OpenAI
```

### Pipeline path

```
GitHub  →  package-<service>  (ci-checks, then SHA tag → ECR)
        →  GitHub Release
        →  release-container-images  (release tag → ECR, Helm-values pull request)
        →  merge to main
        →  Argo CD syncs namespace store
```

GitHub Actions never talks to the cluster. Image tags in Git are the deploy.

### Application services

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Service architecture</b></p>
      <img src="docs/architecture/service-architecture.png" alt="Store service architecture from ingress through the data stores" width="100%">
      <br><sub><b>Figure 2.</b> Ingress → store-front / store-admin → services → DynamoDB, Amazon MQ, DocumentDB, OpenAI.</sub>
    </td>
  </tr>
</table>

| Service | Language | Port | Role |
| --- | --- | --- | --- |
| store-front | Vue | 80 → 8080 | Catalog and cart (`store.simsoliver.com`) |
| store-admin | Vue | 80 → 8081 | Products, orders, Ask AI (`store-admin.simsoliver.com`) |
| product-service | Rust | 3002 | Catalog in DynamoDB `store-products` |
| order-service | Node.js | 3000 | Accepts an order and publishes it to Amazon MQ |
| makeline-service | Go | 3001 | Consumes the queue and writes completed orders to DocumentDB |
| ai-service | Python | 5001 | Product copy through OpenAI |
| virtual-customer | Rust | — | Shopper simulator, 20 orders per hour |
| virtual-worker | Rust | — | Worker simulator, 10 orders per hour |

The six workloads behind a Kubernetes Service autoscale from 3 to 6 replicas at 70% CPU. Pods spread across three availability zones (`maxSkew: 1`, `DoNotSchedule`). The simulators run as three fixed replicas and have no Service.

---

## Design choices

**Two repositories.** Platform changes (a new add-on, a network rule) should not ride along with an application release. Argo CD watches a repository whose job is GitOps.

**A release edits Git, and a merge deploys.** SHA builds prove an image can be produced. Only a release moves the tag that Argo CD will roll out, and a person reviews and merges that pull request.

**CI before the image.** Lint, tests, secret scanning, and SAST gate the publish. GitHub Actions never receives a kubeconfig.

**Managed nodes plus Karpenter.** The managed group is predictable capacity. Karpenter covers bursts and consolidates when the extra nodes are idle.

**Two load balancers.** Customer traffic can take a WAF rule set. Grafana and Argo CD stay off that path.

**One NAT gateway.** Enough for a single environment, and the main reason a forgotten cluster gets expensive. Destroy scripts exist so the account can go back to zero.

**One environment.** Production-shaped controls (private networking, policy, identity, WAF, audit logs) ran on `store-dev`. The account was torn down after the evidence was captured.

---

## Source layout (private repositories)

Application code and platform code live in two private repositories so they can change on different schedules. Argo CD clones only the application repository.

```
store-platform/                         # Terraform, data, add-ons (private)
├── platform/{backend,vpc,acm,eks,karpenter,cert_manager,
│             metrics,monitoring,tempo,logging,otel,argocd}/
├── github_actions_ecr/                 # Private ECR + GitHub OIDC role
├── secrets/                            # Secrets Manager (OpenAI, MQ, DocumentDB)
├── data/{products,documentdb,mq}/
└── scripts-platform/ · scripts-data/

store-app/                              # Services and GitOps (private)
├── src/<service>/                      # Source and Dockerfile
├── helm/                               # Shared chart
├── helm-values/                        # Image tag and config per service
├── argocd/store.yaml                   # ApplicationSet, policies, Ingress
├── k8s/ingress/ · k8s/policies/
├── scripts-store/
└── .github/workflows/                  # ci-checks, package-*, release
```

| Stack order | Creates |
| --- | --- |
| `backend` → `vpc` → `acm` → `eks` | State bucket, network, TLS, cluster |
| `karpenter` → `cert_manager` → `metrics` | Overflow nodes, certs, HPA |
| `monitoring` → `tempo` → `logging` → `otel` | Grafana, traces, CloudWatch, Beyla |
| `argocd` | Argo CD on the ops ALB + Route 53 alias |
| `github_actions_ecr` → `secrets` | ECR repos, OIDC role, Secrets Manager |
| `products` → `documentdb` → `mq` | DynamoDB, DocumentDB, Amazon MQ |

The `simsoliver.com` hosted zone is not created here. This project adds ACM validation and the store aliases only.

---

## CI/CD

A push to `main` builds the service that changed. A GitHub Release is the only path that changes what the cluster runs. GitHub Actions assumes IAM role `store-dev-github-actions-ecr` through OIDC. There is no AWS access key in GitHub.

Every package workflow calls `ci-checks` first. Format, unit tests, Gitleaks, and Semgrep must pass or the image is not published. Dependency audit and Trivy still run and print findings; they do not fail the job, because transitive CVEs change every week. ECR scan-on-push tracks those.

| Workflow | When it runs | What it does |
| --- | --- | --- |
| `ci-checks.yaml` | Called by each package workflow | Format and lint, static analysis, unit tests, dependency audit, Gitleaks, Semgrep, Trivy, smoke, Playwright where the UI has it, Syft SBOM |
| `package-<service>.yaml` | Push or pull request under `src/<service>/` | Runs ci-checks, then builds that image. A push to `main` publishes `store/<service>:<7-char-sha>`. A pull request builds and does not push |
| `package-all-services.yaml` | Manual | Builds all eight SHA tags. Leaves Helm values unchanged |
| `release-container-images.yaml` | GitHub Release published | Builds all eight images as `store/<service>:<release-tag>`, then opens a pull request that sets the image repository and tag in Helm values |

Release `v1.0.1` published all eight images, then opened the Helm-values pull request. The matrix finished in about ten minutes. Merging that pull request is the deploy.

<table>
  <tr>
    <td align="center" width="50%">
      <p align="center"><b>Release queued</b></p>
      <img src="docs/evidence/github-actions-release-queued.png" alt="GitHub Actions release workflow queued" width="100%">
      <br><sub><b>Figure 3.</b> Release workflow queued.</sub>
    </td>
    <td align="center" width="50%">
      <p align="center"><b>Release v1.0.1</b></p>
      <img src="docs/evidence/github-actions-release-success.png" alt="GitHub Actions release v1.0.1 success" width="100%">
      <br><sub><b>Figure 4.</b> Eight images published, then Helm values updated.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <p align="center"><b>Helm-values PR</b></p>
      <img src="docs/evidence/github-pr-helm-values-open.png" alt="Helm-values pull request opened by github-actions" width="100%">
      <br><sub><b>Figure 5.</b> The <code>github-actions</code> bot opened the Helm-values pull request.</sub>
    </td>
    <td align="center" width="50%">
      <p align="center"><b>PR merged</b></p>
      <img src="docs/evidence/github-pr-helm-values-merged.png" alt="Helm-values pull request merged to main" width="100%">
      <br><sub><b>Figure 6.</b> Pull request #4 merged to <code>main</code>.</sub>
    </td>
  </tr>
</table>

### ECR image tags

Tags are immutable. Each repository shows release tag `v1.0.1` and the SHA tags from `main`. Untagged images expire after one day.

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Private ECR</b></p>
      <img src="docs/evidence/ecr-private-repositories.png" alt="Private ECR repositories, one per store service" width="82%">
      <br><sub><b>Figure 7.</b> One private repository per service.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-product-service-tags.png" alt="product-service ECR tags" width="100%">
      <br><sub>product-service</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-order-service-tags.png" alt="order-service ECR tags" width="100%">
      <br><sub>order-service</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-store-front-tags.png" alt="store-front ECR tags" width="100%">
      <br><sub>store-front</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-store-admin-tags.png" alt="store-admin ECR tags" width="100%">
      <br><sub>store-admin</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-makeline-service-tags.png" alt="makeline-service ECR tags" width="100%">
      <br><sub>makeline-service</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-ai-service-tags.png" alt="ai-service ECR tags" width="100%">
      <br><sub>ai-service</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-virtual-customer-tags.png" alt="virtual-customer ECR tags" width="100%">
      <br><sub>virtual-customer</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/ecr-virtual-worker-tags.png" alt="virtual-worker ECR tags" width="100%">
      <br><sub>virtual-worker</sub>
    </td>
  </tr>
</table>

<sub><b>Figure 8.</b> Immutable tags — release <code>v1.0.1</code> and SHA tags from <code>main</code>.</sub>

### GitOps

Terraform installs Argo CD on the ops load balancer at `argocd.simsoliver.com`. Argo CD is not one of the applications it syncs. The DNS alias is a Terraform Route 53 record, not ExternalDNS, so a destroy removes it and a recreate does not keep a leftover hostname.

Sync is automated, with prune and self-heal. Ten applications, all Healthy and Synced at `HEAD`.

| Application | Source | Destination |
| --- | --- | --- |
| ApplicationSet `store` | Chart `helm/` and `helm-values/<name>.yaml` | One Application per service in namespace `store` |
| `store-policies` | `k8s/policies` | NetworkPolicy, ResourceQuota, RBAC, Pod Security Admission labels |
| `store-ingress` | `k8s/ingress` | Store load balancer Ingress |

Queue host, queue URI, and the DocumentDB URI are Terraform outputs, pasted into Helm values before the first sync. Usernames and passwords stay in Secrets Manager. Secrets Store CSI mounts them into the pods.

<table>
  <tr>
    <td align="center" colspan="2">
      <p align="center"><b>Argo CD sign-in</b></p>
      <img src="docs/evidence/argocd-login.png" alt="Argo CD sign-in" width="56%">
      <br><sub><b>Figure 9.</b> Argo CD on the ops load balancer.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <p align="center"><b>Store applications</b></p>
      <img src="docs/evidence/argocd-store-apps.png" alt="Argo CD store applications Healthy and Synced" width="100%">
      <br><sub><b>Figure 10.</b> Store services, ingress, and policies Healthy and Synced.</sub>
    </td>
    <td align="center" width="50%">
      <p align="center"><b>Simulators</b></p>
      <img src="docs/evidence/argocd-store-simulators.png" alt="Argo CD virtual-customer and virtual-worker" width="100%">
      <br><sub><b>Figure 11.</b> virtual-customer and virtual-worker on the same ApplicationSet.</sub>
    </td>
  </tr>
</table>

---

## Infrastructure

Each Terraform folder is its own state. Apply order is the dependency order.

**Platform:** `backend` → `vpc` → `acm` → `eks` → `karpenter` → `cert_manager` → `metrics` → `monitoring` → `tempo` → `logging` → `otel` → `argocd`

**Data and delivery identity:** `github_actions_ecr` → `secrets` → `products` → `documentdb` → `mq`

State is versioned in S3. OpenAI and Slack values stay in local files that Git ignores.

- VPC `10.0.0.0/16` across `us-east-1a`, `us-east-1b`, and `us-east-1c`, with public and private `/24` subnets.
- Nodes and data stores have no public addresses. Egress is one NAT gateway.
- An S3 gateway endpoint carries state, image layers, and provider downloads.
- EKS 1.34. Managed group `store-dev-private-ng`: 3 × `t3.large`, Amazon Linux 2023, encrypted gp3, IMDSv2 required.
- Karpenter NodePool: on-demand `t3` / `t3a` in the same three zones, consolidate when empty or underutilized.
- DynamoDB table `store-products` with point-in-time recovery. Amazon MQ on AMQPS 5671. DocumentDB cluster `store-dev-orderdb`.

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Compute nodes</b></p>
      <img src="docs/evidence/compute-nodes-ready.png" alt="Three t3.large EKS nodes Ready" width="82%">
      <br><sub><b>Figure 12.</b> Three t3.large nodes Ready.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="33%">
      <img src="docs/evidence/pods-platform-addons-a.png" alt="Platform add-ons on the first node" width="100%">
      <br><sub>Add-ons on the first node</sub>
    </td>
    <td align="center" width="33%">
      <img src="docs/evidence/pods-platform-addons-b.png" alt="Platform add-ons on the second node" width="100%">
      <br><sub>Add-ons on the second node</sub>
    </td>
    <td align="center" width="33%">
      <img src="docs/evidence/pods-platform-addons-c.png" alt="ExternalDNS, Pod Identity, and EBS CSI" width="100%">
      <br><sub>ExternalDNS, Pod Identity, and EBS CSI</sub>
    </td>
  </tr>
</table>

<sub><b>Figure 13.</b> Platform add-ons on the nodes.</sub>

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Remote state</b></p>
      <img src="docs/evidence/s3-tfstate-prefixes.png" alt="S3 Terraform state prefixes" width="82%">
      <br><sub><b>Figure 14.</b> Remote state prefixes in S3, one key prefix per stack.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="25%">
      <img src="docs/evidence/s3-tfstate-platform.png" alt="Platform Terraform state keys" width="100%">
      <br><sub>Platform</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/s3-tfstate-data.png" alt="Data store Terraform state" width="100%">
      <br><sub>Data stores</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/s3-tfstate-github-actions-ecr.png" alt="ECR and OIDC Terraform state" width="100%">
      <br><sub>ECR and OIDC</sub>
    </td>
    <td align="center" width="25%">
      <img src="docs/evidence/s3-tfstate-secrets.png" alt="Secrets Manager Terraform state" width="100%">
      <br><sub>Secrets Manager</sub>
    </td>
  </tr>
</table>

<sub><b>Figure 15.</b> Versioned state per stack.</sub>

---

## Application (working run)

The storefront was served at `https://store.simsoliver.com` and the admin UI at `https://store-admin.simsoliver.com`, with TLS on the load balancer. That environment is no longer running. A recruiter opening those hostnames should expect DNS or TLS failure, not a broken project.

<table>
  <tr>
    <td align="center" width="50%">
      <p align="center"><b>Storefront catalog</b></p>
      <img src="docs/evidence/storefront-catalog.png" alt="Storefront catalog" width="100%">
      <br><sub><b>Figure 16.</b> <code>https://store.simsoliver.com</code>.</sub>
    </td>
    <td align="center" width="50%">
      <p align="center"><b>Catalog continued</b></p>
      <img src="docs/evidence/storefront-catalog-continued.png" alt="Storefront catalog continued" width="100%">
      <br><sub><b>Figure 17.</b> Rest of the catalog.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="33%">
      <p align="center"><b>Admin products</b></p>
      <img src="docs/evidence/admin-products.png" alt="Admin product list" width="100%">
      <br><sub><b>Figure 18.</b> Admin product list.</sub>
    </td>
    <td align="center" width="33%">
      <p align="center"><b>Ask AI</b></p>
      <img src="docs/evidence/admin-ask-ai-generating.png" alt="Ask AI generating a product description" width="100%">
      <br><sub><b>Figure 19.</b> Ask AI in progress.</sub>
    </td>
    <td align="center" width="33%">
      <p align="center"><b>Description written</b></p>
      <img src="docs/evidence/admin-ask-ai-generated.png" alt="Ask AI description written back onto the product form" width="100%">
      <br><sub><b>Figure 20.</b> Description written back onto the form.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="33%">
      <p align="center"><b>Orders</b></p>
      <img src="docs/evidence/admin-orders.png" alt="Admin orders including simulator traffic" width="100%">
      <br><sub><b>Figure 21.</b> Admin orders, including simulator traffic.</sub>
    </td>
    <td align="center" width="33%">
      <p align="center"><b>Order detail</b></p>
      <img src="docs/evidence/admin-order-detail.png" alt="Admin order detail" width="100%">
      <br><sub><b>Figure 22.</b> Order detail.</sub>
    </td>
    <td align="center" width="33%">
      <p align="center"><b>Order completed</b></p>
      <img src="docs/evidence/admin-order-completed.png" alt="Admin order completed" width="100%">
      <br><sub><b>Figure 23.</b> Order completed.</sub>
    </td>
  </tr>
</table>

---

## Security snapshot

These controls were on the cluster during validation.

| Control | Implementation |
| --- | --- |
| Secrets | AWS Secrets Manager · Secrets Store CSI · no secret files in Git |
| CI identity | GitHub OIDC to `store-dev-github-actions-ecr` · no long-lived key |
| Workload identity | EKS Pod Identity · instance metadata `169.254.169.254` blocked |
| Pod security | PSA `restricted` · non-root · drop `ALL` · read-only root · `RuntimeDefault` seccomp |
| Network | Private subnets · default-deny NetworkPolicy in `store` · per-service ingress |
| Edge | WAF `AWSManagedRulesCommonRuleSet` on the store ALB · ops ALB for Grafana and Argo CD |
| Images | Private ECR · immutable tags · scan on push · Gitleaks, Semgrep, Trivy in CI |
| Nodes | IMDSv2 required · encrypted EBS · Karpenter hop limit 1 |
| Audit | EKS control-plane logs including audit · 7-day retention |
| Quotas | ResourceQuota on namespace `store` |

---

## Observability

Add-ons are Terraform Helm releases. They are not part of the store ApplicationSet, so an application sync cannot remove Grafana or the log pipeline.

| Signal | Path |
| --- | --- |
| Metrics | kube-prometheus-stack · Grafana at `grafana.simsoliver.com` |
| Logs | Fluent Bit → CloudWatch · one group per service · 7-day retention |
| Traces | Grafana Beyla (eBPF) → OpenTelemetry Collector → Tempo · no SDK in the services |
| Alerts | Alertmanager → Slack `#devops-alerts` |

Storefront `POST /api/orders` traces from live checkout were in the 5–12 ms range. `ai-service` `POST /generate/description` ran about 26–30 seconds, which matches the OpenAI call behind Ask AI.

<table>
  <tr>
    <td align="center" colspan="2">
      <p align="center"><b>Grafana sign-in</b></p>
      <img src="docs/evidence/grafana-login.png" alt="Grafana sign-in" width="56%">
      <br><sub><b>Figure 24.</b> Grafana on the ops load balancer.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <img src="docs/evidence/grafana-store-cpu-memory.png" alt="Grafana CPU and memory in namespace store" width="100%">
      <br><sub>CPU and memory while orders were flowing</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/evidence/grafana-store-memory-network.png" alt="Grafana memory and network" width="100%">
      <br><sub>Memory and network</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <img src="docs/evidence/grafana-store-pod-network.png" alt="Grafana pod network" width="100%">
      <br><sub>Pod network</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/evidence/grafana-store-io-drops.png" alt="Grafana drops and disk IO" width="100%">
      <br><sub>Drops and disk IO</sub>
    </td>
  </tr>
</table>

<sub><b>Figure 25.</b> Grafana dashboards in namespace <code>store</code>.</sub>

<table>
  <tr>
    <td align="center">
      <p align="center"><b>CloudWatch log groups</b></p>
      <img src="docs/evidence/cloudwatch-log-groups.png" alt="CloudWatch log groups for store services" width="82%">
      <br><sub><b>Figure 26.</b> One group per service, plus the control plane and Amazon MQ.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-virtual-customer-logs.png" alt="virtual-customer logs" width="100%"><br><sub>virtual-customer</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-virtual-worker-logs.png" alt="virtual-worker logs" width="100%"><br><sub>virtual-worker</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-ai-service-logs.png" alt="ai-service logs" width="100%"><br><sub>ai-service</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-makeline-service-logs.png" alt="makeline-service logs" width="100%"><br><sub>makeline-service</sub></td>
  </tr>
  <tr>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-order-service-logs.png" alt="order-service logs" width="100%"><br><sub>order-service</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-product-service-logs.png" alt="product-service logs" width="100%"><br><sub>product-service</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-store-front-logs.png" alt="store-front logs" width="100%"><br><sub>store-front</sub></td>
    <td align="center" width="25%"><img src="docs/evidence/cloudwatch-store-admin-logs.png" alt="store-admin logs" width="100%"><br><sub>store-admin</sub></td>
  </tr>
</table>

<sub><b>Figure 27.</b> Per-service CloudWatch logs.</sub>

<table>
  <tr>
    <td align="center">
      <p align="center"><b>Tempo · store-front POST /api/orders</b></p>
      <img src="docs/evidence/tempo-store-front-orders.png" alt="Tempo trace for store-front POST /api/orders" width="82%">
      <br><sub><b>Figure 28.</b> Storefront checkout trace.</sub>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center" width="33%"><img src="docs/evidence/tempo-store-admin-products.png" alt="Tempo store-admin products" width="100%"><br><sub>store-admin</sub></td>
    <td align="center" width="33%"><img src="docs/evidence/tempo-product-service-get.png" alt="Tempo product-service" width="100%"><br><sub>product-service</sub></td>
    <td align="center" width="33%"><img src="docs/evidence/tempo-order-service-post.png" alt="Tempo order-service" width="100%"><br><sub>order-service</sub></td>
  </tr>
  <tr>
    <td align="center" width="33%"><img src="docs/evidence/tempo-makeline-order-fetch.png" alt="Tempo makeline fetch" width="100%"><br><sub>makeline</sub></td>
    <td align="center" width="33%"><img src="docs/evidence/tempo-ai-service-description.png" alt="Tempo ai-service description" width="100%"><br><sub>ai-service</sub></td>
    <td align="center" width="33%"><img src="docs/evidence/tempo-virtual-customer-post.png" alt="Tempo virtual-customer" width="100%"><br><sub>virtual-customer</sub></td>
  </tr>
</table>

<table>
  <tr>
    <td align="center">
      <img src="docs/evidence/tempo-virtual-worker-fetch.png" alt="Tempo virtual-worker" width="56%">
      <br><sub>virtual-worker</sub>
    </td>
  </tr>
</table>

<sub><b>Figure 29.</b> Beyla HTTP traces in Tempo.</sub>

<table>
  <tr>
    <td align="center" width="50%">
      <p align="center"><b>Slack</b></p>
      <img src="docs/evidence/slack-signin.png" alt="Slack workspace sign-in" width="100%">
      <br><sub><b>Figure 30.</b> Slack workspace that receives the alerts.</sub>
    </td>
    <td align="center" width="50%">
      <p align="center"><b>Alertmanager</b></p>
      <img src="docs/evidence/slack-devops-alerts.png" alt="Alertmanager posting to Slack" width="100%">
      <br><sub><b>Figure 31.</b> Pod, CPU, HPA, and quota alerts in <code>#devops-alerts</code>.</sub>
    </td>
  </tr>
</table>

---

## Operations

```
platform:  backend → vpc → acm → eks → karpenter → cert_manager
           → metrics → monitoring → tempo → logging → otel → argocd
data:      github_actions_ecr → secrets → products → documentdb → mq
store:     namespace, CSI classes, Argo CD applications
```

Bring-up is platform, then data, then store. Teardown is the reverse. Namespace `store` has to be gone, and Karpenter nodes drained, before the cluster is destroyed.

The private repositories include ordered create, check, and destroy scripts, plus runbooks. Passwords, personal access tokens, and API keys stay out of Git.

Hostnames from the validated run:

| Use | Hostname |
| --- | --- |
| Storefront | `store.simsoliver.com` |
| Admin | `store-admin.simsoliver.com` |
| Grafana | `grafana.simsoliver.com` |
| Argo CD | `argocd.simsoliver.com` |

---

Oliver Sims
