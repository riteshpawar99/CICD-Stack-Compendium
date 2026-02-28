# 🛠️ CI/CD Stack Compendium — Four Production Pipelines, Zero Guesswork

> A comprehensive, interactive reference documenting four real-world CI/CD stacks in complete, stage-by-stage detail — with actual configuration snippets, exact tooling decisions, and the reasoning behind every choice.

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![GitLab CI](https://img.shields.io/badge/GitLab_CI-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Azure DevOps](https://img.shields.io/badge/Azure_DevOps-0078D7?style=for-the-badge&logo=azure-devops&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)

---

## 📌 What This Project Is

This is a production-grade reference guide that maps four complete, distinct CI/CD pipelines — **GitHub Actions + AWS**, **GitLab CI + GCP**, **Jenkins + On-Premise**, and **Azure DevOps** — in exhaustive detail. Every stage of every pipeline is documented with exact tools, the reasoning for choosing those tools over alternatives, and real, copy-paste-ready configuration snippets.

The interactive interface was built from scratch using semantic HTML5, modern CSS (custom properties, grid, animations), and vanilla JavaScript — no external frameworks, no dependencies, no build step. Every byte of the UI was written by hand.

This project is not a surface-level comparison of CI/CD platforms. It is a demonstration that I understand these systems deeply enough to choose between them intelligently and implement them correctly.

---

## 🧠 What Motivated This & What It Shows

I built this project because I hit a specific gap in my learning: most CI/CD resources either teach you one platform in isolation, or provide a vague high-level comparison without ever getting into the mechanics of *how* you actually configure and run each stage. The result is a kind of shallow familiarity — you know the names of the tools, but not why you'd pick one over another, or what configuration decisions matter in practice.

I wanted to close that gap. So I researched and implemented the four most common production stacks used by real engineering teams, documented every stage with actual YAML and Groovy configuration, and built an interface that makes the whole thing navigable and readable.

This project demonstrates that I understand:

- Why different organizations choose different CI/CD stacks based on their cloud provider, compliance requirements, team size, and existing tooling investment
- The conceptual differences between push-based and pull-based (GitOps) deployment models, and why the industry is moving toward GitOps
- How to read, write, and reason about pipeline configuration in four different formats: GitHub Actions YAML, GitLab CI YAML, Jenkins Declarative Pipelines (Groovy DSL), and Azure Pipelines YAML
- What "shift-left security" actually means in practice — not just as a buzzword, but as a specific set of scanning tools integrated at specific pipeline stages
- The difference between a registry, a repository, and a release artifact — and why the tagging and signing strategy you use matters for traceability and supply chain security
- How secrets management works correctly (and incorrectly) in CI/CD environments

---

## 🏗️ The Four Stacks — Architecture & Rationale

### Stack 1 — GitHub Actions + AWS

**Who uses this:** Startups, modern SaaS companies, open-source projects, and teams that live in the GitHub ecosystem.

**Why this combination makes sense:** GitHub Actions is tightly integrated with the GitHub source repository, which means pipeline triggers, pull request status checks, and branch protection rules all work as a single cohesive system. You don't need to configure webhooks or authenticate a separate CI server to your repo — the pipeline YAML lives in `.github/workflows/` and GitHub handles everything. AWS is the dominant cloud provider, so pairing GitHub Actions with ECR (Elastic Container Registry) and EKS (Elastic Kubernetes Service) is the most common production setup for this tier.

**The workflow I documented, stage by stage:**

The **source stage** establishes the trigger mechanism. The `on: push` and `on: pull_request` event declarations in the YAML tell GitHub which git events should fire the pipeline. Branch protection rules add a layer of social and technical enforcement — no code merges to `main` without the pipeline passing, regardless of who the author is. I also documented Conventional Commits here because it's an underrated practice: when your commit messages follow a consistent format (`feat:`, `fix:`, `chore:`), you can automate changelog generation and semantic version bumping as a pipeline step rather than a manual decision.

The **build stage** uses `actions/cache` to cache the `~/.npm` directory between runs, keyed on the hash of `package-lock.json`. This means if your dependencies haven't changed, the install step takes seconds instead of minutes. The Docker build uses BuildKit's `--cache-from` flag to reuse layers from the previous image — on large projects, this can cut build times by 60-70%.

The **test stage** uses a `services:` block in the GitHub Actions job to spin up a real Postgres container and a real Redis container as sidecar services during testing. This is important: your integration tests should run against real infrastructure, not mocks, because mocks can't catch behavioral differences between your assumptions and what the real service actually does.

The **security stage** implements four distinct scan types because they catch four distinct threat surfaces that don't overlap. CodeQL catches logic-level vulnerabilities in your source code (the kind that OWASP classifies as A1–A10). Trivy catches known CVEs in your Docker image's OS packages and application dependencies. gitleaks scans the full git history for secrets — not just the current commit, but every commit ever made, because secrets committed and then "deleted" still exist in history and can be extracted by anyone with repository access.

The **artifact stage** introduces image signing with `cosign`, which is worth explaining carefully because it's often skipped by less experienced engineers. Signing a Docker image creates a cryptographic attestation — a verifiable claim that says "this specific image (identified by its immutable digest) was produced by this CI pipeline running in this repository." Any admission controller on your Kubernetes cluster can then verify this signature before allowing the image to run. Without signing, there's no way to prove that the image in your registry is the same one your pipeline built — it could have been overwritten, tampered with, or replaced.

The **deploy stage** implements the GitOps pattern via ArgoCD and deserves the most careful explanation because it represents a fundamentally different mental model from traditional deployment. In a traditional CI/CD pipeline, the CI system (GitHub Actions, Jenkins) has credentials to talk to the Kubernetes cluster and runs `kubectl apply` or `helm upgrade` directly. This is "push-based" deployment. It works, but it has a critical flaw: the cluster can drift from your configuration without anyone noticing, because nothing is continuously reconciling the cluster's actual state against a declared desired state.

In a GitOps model, the flow is inverted. Your CI pipeline does *not* talk to the cluster. Instead, it updates the image tag in a separate "config repository" and pushes that change. ArgoCD, running inside the cluster, continuously polls the config repository and compares what's in git to what's actually running. When it sees a difference, it reconciles — bringing the cluster into alignment with the git state. The cluster *pulls* its desired state from git. This means git is your single source of truth for every cluster at every point in time, rollback is a `git revert`, and auditing is a `git log`.

The **canary deployment strategy** I documented via Argo Rollouts deserves particular attention. A canary release sends only a small percentage of real production traffic (I used 10% as the initial slice) to the new version while the rest continues to receive the proven, stable version. An analysis template continuously monitors error rates and latency. If either metric crosses a threshold, the rollout is automatically aborted and all traffic is returned to the old version — without a human ever needing to intervene. This is not just about safety; it's about being able to deploy at any time of day without fear, because the system will protect you if something goes wrong.

---

### Stack 2 — GitLab CI + GCP

**Who uses this:** Larger enterprise dev teams who want a single-vendor DevSecOps platform, organizations standardized on Google Cloud, and teams that need built-in security scanning without purchasing additional tools.

**Why this combination is different:** GitLab's biggest differentiator is that security scanning is a first-class, built-in feature rather than a collection of third-party Actions you have to stitch together. By including a single line — `include: template: Security/SAST.gitlab-ci.yml` — you activate a scanning suite that runs 15+ language-specific analyzers (Semgrep for JS/Python, Bandit for Python, SpotBugs for Java, and more), reports findings directly in the Merge Request UI, and gates merges on security findings. For enterprise teams with compliance requirements, this is a significant advantage because the toolchain is auditable and consistent across every project in the organization.

**The most important technical concept I documented here — Kaniko:** When your CI runner is a pod in a Kubernetes cluster (which GitLab runners commonly are), you have a problem: building a Docker image normally requires access to a Docker daemon, and running a Docker daemon inside a container requires privileged mode, which is a significant security risk in a shared cluster. Kaniko solves this by building Docker images from a Dockerfile entirely in userspace, without needing a daemon or privileged access at all. It reads each instruction in the Dockerfile, executes it in an unprivileged container, and snapshots the filesystem changes — producing a valid OCI image without ever needing root. This is not a minor detail; it's a production requirement for secure Kubernetes-based CI infrastructure.

**Flux CD vs ArgoCD:** For the GCP stack I documented Flux as the GitOps tool rather than ArgoCD, because this illustrates an important point: GitOps is a pattern, not a specific tool. Both ArgoCD and Flux implement the GitOps pattern, but with architectural differences. ArgoCD has a richer UI and is often preferred by teams that want visibility into sync status across many applications. Flux is more lightweight, integrates more naturally with Helm and Kustomize, and is particularly strong for multi-cluster management via fleet-level configuration. The `ImagePolicy` resource I showed is a Flux-specific feature that automates image tag updates: when a new image is pushed to the registry, Flux detects it, updates the manifest in git (automatically committing the new tag), and then syncs the cluster — creating a fully automated path from `docker push` to running pod.

---

### Stack 3 — Jenkins + On-Premise

**Who uses this:** Regulated industries (banking, healthcare, government), organizations with data residency requirements, air-gapped environments, and enterprises with large existing Jenkins investments.

**Why Jenkins persists despite being older:** Jenkins is not the "legacy choice" — it's the *control choice*. Every other CI/CD system makes trade-offs: they're managed services, which means you don't control where builds run, what data they can access, or how the platform itself is secured and updated. For organizations subject to PCI-DSS, HIPAA, or government security frameworks, the ability to run your entire CI/CD system on hardware you own and operate, behind a firewall you control, with no data ever leaving your network, is not a nice-to-have. It's a compliance requirement.

**The Shared Library pattern I documented is the most important Jenkins concept for scale:** Without it, you end up with every team writing their own Jenkinsfile from scratch, and you get 40 different versions of "how we do deployments" across 40 repositories. A Shared Library is a separate git repository containing reusable Groovy functions that any Jenkinsfile in your organization can import with `@Library('pipeline-lib') _`. Your platform engineering team owns the Shared Library — they define the `deployToK8s()` function, the `sonarScan()` function, the `slackNotify()` function. Application teams call these functions without needing to understand their implementation. This is the DRY principle applied to pipeline infrastructure.

**Secrets management with HashiCorp Vault:** In cloud-native stacks, secrets management is relatively straightforward — AWS Secrets Manager, GCP Secret Manager, and Azure Key Vault are managed services that integrate naturally with their respective CI/CD platforms via short-lived OIDC tokens. On-premise, you need to provide this yourself. HashiCorp Vault is the standard solution. The `withVault()` block I documented shows the correct pattern: secrets are never stored in Jenkinsfile, environment variables, or job configurations. Instead, the pipeline requests a specific secret from Vault at runtime, Vault authenticates the Jenkins job via its AppRole or Kubernetes auth method, and the secret is injected as an environment variable for the duration of that step only. It's then garbage-collected. No secret ever persists in the build environment.

**Manual approval gates:** The `input{}` block I included is a Jenkins feature that pauses pipeline execution and sends a notification (Slack, email) asking a named individual to approve continuation. This is not a workaround or a limitation — for regulated deployments (a financial institution pushing to a production trading system, for instance), a documented human approval is a compliance requirement. The approver's identity, timestamp, and decision are captured in Jenkins's build log, creating an immutable audit trail.

---

### Stack 4 — Azure DevOps

**Who uses this:** Organizations in the Microsoft ecosystem — .NET shops, Windows workloads, companies with existing Azure subscriptions, enterprises using Microsoft 365 and needing seamless integration across their toolchain.

**Why Azure DevOps is cohesive in a way other stacks aren't:** When you're using Azure Repos, Azure Pipelines, Azure Artifacts, Azure Container Registry, AKS, and Azure Monitor together, they're all integrated under a single identity and access management system (Azure Active Directory / Entra ID). You don't need to configure service principals and credentials between each pair of services — a Service Connection established once in Azure DevOps grants scoped access across the whole platform. For IT organizations that already manage user identity through Active Directory, this single-pane-of-glass approach reduces operational overhead significantly.

**Azure Environments and the Approval pattern:** One of the most production-relevant features I documented is the **Environment resource** in Azure Pipelines. When a pipeline stage targets a named Environment (like `production`), Azure DevOps allows you to attach multiple types of checks to that environment: manual approval by specific individuals or groups, a branch control check (only deployments from `main` can target production), a business hours check (no deploys on weekends), or an invocation of an external REST API for a custom gate. Crucially, the approval is logged with the approver's Azure AD identity and timestamp in the pipeline run history. This is how you implement a compliant change management process in code — no spreadsheets, no ticketing system integration required.

**Bicep for infrastructure as code:** Rather than documenting Terraform (which works but is a third-party tool), I highlighted **Bicep** because it's Azure-native. Bicep is a domain-specific language that compiles down to ARM (Azure Resource Manager) templates, meaning it has first-class support for every Azure resource type from day one, without waiting for a Terraform provider to be updated. For pure Azure shops, Bicep is the most direct path from "I want a Kubernetes cluster" to a declarative, version-controlled, reviewable infrastructure definition.

---

## 📊 Choosing the Right Stack — Decision Framework

Choosing a CI/CD stack is not a purely technical decision. It is shaped by your organization's existing infrastructure, compliance requirements, cloud commitments, and team expertise. Here is how I would frame the decision:

**Start with GitHub Actions + AWS** if you're building a new product from scratch, your team is comfortable in the GitHub ecosystem, and you don't have specific on-prem or vendor requirements. This stack has the lowest operational overhead, the best developer experience, and the most extensive ecosystem of pre-built Actions.

**Move to GitLab + GCP** if your security and compliance posture requires integrated DevSecOps tooling, you need self-hosting capability, or your team is standardized on GCP. GitLab's all-in-one approach reduces the number of vendors and integrations you need to manage.

**Choose Jenkins + On-Premise** only when you have genuine requirements that prevent cloud CI/CD — air-gap requirements, data residency rules, or regulatory frameworks that mandate on-premises processing. Jenkins is powerful and flexible, but it carries real operational overhead. Don't choose it for nostalgia or familiarity.

**Choose Azure DevOps** if your organization is already invested in the Microsoft ecosystem, your developers are on Windows, your applications target .NET or Azure-specific services, and your IT governance is built around Azure Active Directory.

---

## 🛠️ Complete Toolchain Matrix

The table below maps the complete toolchain across all four stacks, stage by stage. Reading across a row shows you which tool in each stack serves the same purpose — understanding these equivalences is what allows you to move between stacks without starting from zero.

| Stage | GitHub + AWS | GitLab + GCP | Jenkins + On-Prem | Azure DevOps |
|---|---|---|---|---|
| **Source** | GitHub | GitLab | Bitbucket / GitHub | Azure Repos |
| **CI Engine** | GitHub Actions | GitLab CI | Jenkins (K8s executor) | Azure Pipelines |
| **Build** | ubuntu-latest runner | GitLab Runner + Kaniko | Jenkins Agent Pod | Microsoft-hosted agent |
| **Unit Test** | Jest / pytest | pytest / pytest | JUnit / Jest | pytest / xUnit |
| **Integration Test** | Testcontainers | Docker Compose services | Testcontainers | Testcontainers |
| **SAST** | CodeQL | GitLab SAST (built-in) | SonarQube / Checkmarx | Microsoft Defender for DevOps |
| **Dependency Scan** | Snyk / Dependabot | GitLab Dependency Scanning | OWASP Dependency-Check | Microsoft Defender for DevOps |
| **Container Scan** | Trivy | GitLab Container Scanning | Trivy | Microsoft Defender for Containers |
| **Secret Detection** | gitleaks | GitLab Secret Detection | truffleHog | Microsoft Defender for DevOps |
| **Registry** | Amazon ECR | Google Artifact Registry | Nexus / Artifactory | Azure Container Registry |
| **Image Signing** | cosign (Sigstore) | cosign (Sigstore) | cosign / Notary | Azure Image Integrity |
| **IaC** | Terraform / CDK | Terraform | Terraform / Ansible | Bicep / Terraform |
| **Secrets** | AWS Secrets Manager | Google Secret Manager | HashiCorp Vault | Azure Key Vault |
| **GitOps Engine** | ArgoCD | Flux CD | ArgoCD / Spinnaker | Azure GitOps (Flux) |
| **Kubernetes** | Amazon EKS | Google GKE | On-prem K8s / OpenShift | Azure AKS |
| **Deploy Strategy** | Argo Rollouts (canary) | Cloud Deploy (progressive) | Ansible / Spinnaker | Deployment strategy YAML |
| **Logs** | CloudWatch Logs | Google Cloud Logging | ELK Stack | Azure Monitor Logs |
| **Metrics** | Prometheus + CloudWatch | Google Cloud Monitoring | Prometheus + Grafana | Azure Monitor Metrics |
| **Tracing** | OpenTelemetry → Datadog | OpenTelemetry → Cloud Trace | Jaeger / Zipkin | Application Insights |
| **Alerting** | PagerDuty / OpsGenie | PagerDuty / Google Alerting | Alertmanager → PagerDuty | Azure Monitor Alerts |
| **On-Call** | PagerDuty | PagerDuty | PagerDuty / OpsGenie | Azure DevOps On-call |

---

## 💡 Deep Concepts I Internalized Building This

**The difference between CI and CD matters more than the acronym suggests.** CI (Continuous Integration) ends at a tested, validated, published artifact. CD can mean two different things: Continuous Delivery (the artifact is always *ready* to deploy, but a human triggers the actual release) or Continuous Deployment (every passing build is automatically deployed to production with no human gate). Which you need depends on your product, team maturity, and stakeholder tolerance for risk.

**Pipeline configuration is application code and should be treated like it.** Pipeline YAML files have their own bugs, security vulnerabilities, and drift problems. They should live in version control (they do, because they're in your repo), be reviewed in pull requests, and — at sufficient organizational scale — be generated from higher-level abstractions (like a Platform Engineering team's Shared Library or a templating system) rather than hand-written per-repo.

**The registry is a trust boundary, not just storage.** What your production cluster is willing to run should be governed by policy. Enforcing that only signed images can be deployed, that images must have passed a container scan, or that images must come from your organization's registry (not from Docker Hub or a random third-party source) is what supply chain security means at the Kubernetes layer.

**Observability is not monitoring.** Monitoring tells you when something is broken (alert: error rate > 5%). Observability lets you understand *why* it is broken, using the data your system emits — logs, metrics, and traces together. Building observability in from the start (instrumenting with OpenTelemetry, structuring logs as JSON rather than plain text, tagging every metric with environment and version) is orders of magnitude cheaper than retrofitting it after an incident.

**Idempotency is a pipeline superpower.** A good pipeline can be run multiple times on the same commit and produce the same result. Lockfiles for dependencies, pinned base images in Dockerfiles, Terraform with remote state, Helm charts with fixed values — all of these contribute to a deterministic pipeline that builds the same artifact on Tuesday as it did last Thursday. This makes debugging failures dramatically easier, because you can eliminate "environment drift" as a variable.

---

## 🚀 How to View & Use This Reference

```bash
# Clone the repository
git clone https://github.com/yourusername/cicd-stack-reference.git

# Open the interactive reference
open index.html       # macOS
xdg-open index.html  # Linux
start index.html      # Windows
```

Use the four tab buttons at the top to switch between stacks. Each tab shows a complete, independent pipeline — stages, tools, reasoning, and configuration — for that specific platform combination.

---

## 📁 Project Structure

```
cicd-stack-reference/
├── index.html          # Complete interactive multi-stack reference
├── README.md           # This file
└── assets/
    └── preview.png     # (optional) screenshot for README header
```

---

## 🔮 Roadmap — From Reference to Reality

This project is a reference implementation. The natural progression from here, in order of complexity and learning value:

**Phase 1 — Hands-on GitHub Actions:** Build a real `.github/workflows/ci.yml` that implements every stage documented in Stack 1. Use a real Node.js or Python application. Get it running in GitHub's actual CI infrastructure.

**Phase 2 — Local GitOps lab:** Spin up a local Kubernetes cluster with `k3d` or `kind`. Install ArgoCD. Create a config repository. Push image tag updates and watch ArgoCD reconcile the cluster. This makes the GitOps pattern visceral in a way that reading about it cannot.

**Phase 3 — Observability end-to-end:** Instrument a sample application with OpenTelemetry, run a local Jaeger or Grafana Tempo collector, generate some test traffic, and trace a request through multiple services. Understanding traces visually changes how you think about distributed systems debugging.

**Phase 4 — Jenkins hands-on:** Spin up a local Jenkins instance via Docker, configure a Kubernetes executor, write a Shared Library, and build a Jenkinsfile that calls it. This is the hardest stack to get hands-on with from zero — but if you can build it locally, you can work with any Jenkins installation.

---

## 📚 References & Deep Reading

Every tool documented in this reference has extensive official documentation. Here are the starting points that shaped this project most:

- [GitHub Actions documentation](https://docs.github.com/en/actions) — the definitive reference for workflow syntax and available actions
- [ArgoCD Getting Started Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/) — the best hands-on introduction to GitOps in practice
- [Flux CD documentation](https://fluxcd.io/flux/) — particularly the ImageAutomation section for understanding automated tag updates
- [GitLab CI/CD documentation](https://docs.gitlab.com/ee/ci/) — especially the Security section on built-in scanning templates
- [Jenkins Pipeline documentation](https://www.jenkins.io/doc/book/pipeline/) — the Declarative Pipeline syntax reference
- [Kaniko documentation](https://github.com/GoogleContainerTools/kaniko) — understanding rootless image builds
- [Sigstore / cosign](https://docs.sigstore.dev/) — software supply chain security and image signing
- [OpenTelemetry documentation](https://opentelemetry.io/docs/) — the instrumentation specification and SDK guides
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs) — particularly the auth methods section for CI/CD integration
- [DORA research](https://dora.dev/research/) — the foundational research behind measuring DevOps performance

---

## 👤 Author

Built as a deep-dive learning project to move from knowing CI/CD by name to understanding it by design. Every tool listed, every configuration snippet shown, every architectural decision explained — researched and understood before being written down.

-RITESSH PAWAR 
riteshpawar754@gmail.com
---

*"There is no universally correct CI/CD stack. There is only the stack that fits your team's constraints, cloud commitments, and compliance requirements — and that your engineers understand well enough to debug at 2am when something goes wrong."*
