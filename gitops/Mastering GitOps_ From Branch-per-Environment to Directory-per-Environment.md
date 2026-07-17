### Mastering GitOps: From Branch-per-Environment to Directory-per-Environment

Modern software delivery demands a rigorous separation between the application’s executable logic and its operational state. As systems scale across Amazon EKS and legacy on-premises data centers, traditional branching strategies often collapse into "deployment hell." This guide outlines the transition to a modern, directory-based GitOps architecture designed for Java-based enterprise environments.

#### 1\. The Fundamental Shift: Separating State from Binary

The "12-Factor App" philosophy dictates a strict isolation of application code from its configuration. In a Java ecosystem, we distinguish between the  **Binary**  (the immutable artifact) and the  **State**  (the environment-specific settings).

##### Asset Comparison Matrix

Asset Type,Storage Location,"The ""Why"" (Architectural Logic)"  
Environment Configs,Git,"Text-based files (YAML, .properties) require version control and PR-based auditing.  JFrog Artifactory does not handle text-diffs well."  
Docker Images,JFrog Artifactory,"Heavy, immutable binary blobs. Artifactory is optimized for vulnerability scanning and serving large layers to EKS or On-Prem nodes."  
Compiled Code,JFrog Artifactory,Raw .jar or .war files are stored here for shared dependencies  before  being wrapped in an environment-agnostic Docker image.

##### The 3 Critical Reasons Why Config is Never Bundled

1. **Auditability & Text-Diffs:**  Storing configuration in Git provides a clear trail of  **who**  changed  **which**  line and  **why** . Since JFrog is a binary repository, it cannot provide the granular line-by-line audit required for configuration compliance.  
2. **Immutability:**  A Docker image should be built once and promoted through every environment. If configuration is bundled inside, you must rebuild the entire image just to update a database timeout, violating the principle that the code tested in QA is exactly what runs in Prod.  
3. **The "Single Source of Truth" Paradox:**  Git defines "how the environment should behave," while Artifactory defines "the code that executes." Bundling them merges these concerns, making it impossible to update environmental state without a new code release.This separation of concerns necessitates a fundamental rethink of how we utilize Git branching to manage infrastructure.

#### 2\. The Legacy Model: The Pitfalls of Branch-per-Environment

In the legacy model, teams often create dedicated, long-lived branches (e.g., dev, qa, prod) that map directly to environments. While intuitive, this approach is an anti-pattern that creates technical debt.

##### Primary Anti-Patterns and Production Risks

* **Merge Conflicts (The Hotfix Trap):**  If a critical hotfix is applied directly to the prod branch, the branches diverge. Future merges from qa to prod risk overwriting that fix or resulting in complex conflicts that stall the release pipeline.  
* **Environment Drift:**  Over time, environment-specific branches accumulate small, unmerged differences. This ensures that the QA environment eventually ceases to accurately mirror Production, leading to the dreaded "worked in QA, failed in Prod" scenario.  
* **Lack of Visibility:**  There is no unified view of the entire infrastructure. An architect must switch between three or four branches to understand the global state of the system at any given moment.To solve these pain points, modern DevOps has shifted toward a model that uses a single branch to represent the entire infrastructure state.

#### 3\. The Modern Standard: Directory-per-Environment on a Single Branch

The modern standard uses a single main branch as the  **Single Source of Truth** . Instead of using branches to separate environments, we use directories. This allows for concurrent releases and "Ephemeral Environments."

##### Comparison: Branch-Per-Environment vs. Directory-Per-Environment

Feature,Branch-Per-Environment,Directory-Per-Environment  
Source of Truth,Scattered across multiple branches.,Unified in the single main branch.  
Promoting Changes,Risky Git merges between branches.,Simple file updates or PRs within directories.  
Drift Prevention,Difficult; branches naturally diverge.,"Easy; shared ""base"" configs enforce consistency."  
Tool Compatibility,Can confuse GitOps operators.,Natively expected by tools like Argo CD.

##### Recommended Directory Structure (Repo B)

By organizing by directory, you can manage multiple platforms and concurrent releases simultaneously:  
deploy/                       \# The GitOps Configuration Root  
├── k8s-eks/                  \# Target for Argo CD  
│   ├── base/                 \# Common deployments and services  
│   └── overlays/             \# Environment-specific overrides  
│       ├── qa-v1.2/          \# Ephemeral Env for v1.2 testing  
│       ├── qa-v1.3/          \# Concurrent Ephemeral Env for v1.3  
│       └── prod/             \# Production ConfigMaps and image tags  
└── on-prem/                  \# Target for GitHub Actions  
    ├── docker-compose.yml    \# Base compose referencing JFrog images  
    ├── qa/                   \# QA application.yml and .env  
    └── prod/                 \# Prod application.yml and .env

To prevent versioning confusion, we further decouple the application code from these deployment manifests into a two-repository architecture.

#### 4\. The Two-Repository Architecture: Application vs. Deployment

Managing Java code and deployment manifests in the same repository creates a  **Versioning Paradox** : the code on main may be at v1.4, while the production config sitting next to it is at v1.1. The solution is the  **Two-Repo Strategy** .

##### Responsibility Matrix

Repository,Contents,Purpose,Security & Access  
Repo A (App),"Java code, pom.xml, Dockerfile.",Building the immutable artifact.,Open to all 20+ Developers for PRs.  
Repo B (GitOps),"K8s manifests, Docker Compose, Env Configs.",Managing environment state.,Restricted to Tech Leads/DevOps.

##### Developer Experience: The Local Autonomy Profile

To ensure developers can work efficiently without Repo B access, Repo A remains self-contained for local work. Developers use  **Spring Boot Profiles**  and a local configuration:

* **application-local.yml** : Contains mock service URLs and localhost database strings.  
* **Execution** : Developers run with \-Dspring.profiles.active=local.  
* **Local Deps** : Repo A includes a docker-compose-local.yml or uses  **Testcontainers**  to spin up local PostgreSQL/Kafka instances in IntelliJ.This architecture ensures that the application build and the environment deployment are decoupled but interact seamlessly during a live release.

#### 5\. Promotion and Workflow: EKS vs. On-Premises

The synchronization mechanism depends on the target infrastructure, but both follow GitOps principles.

##### Argo CD Workflow (EKS)

1. **Sync:**  Argo CD monitors deploy/k8s-eks/overlays/ in Repo B. When a PR is merged, it applies the manifests.  
2. **ConfigMap Detection:**   **Architectural Note:**  Java pods do not natively detect ConfigMap changes.  
3. **Pro Tip (Stakater Reloader):**  We install the Stakater Reloader operator. It watches for  **ConfigMap updates**  specifically and triggers a rolling restart of the Java pods to ensure the new configuration is injected.

##### GitHub Actions Workflow (On-Prem)

1. **Filter:**  Triggers only on changes to deploy/on-prem/prod/\*\*.  
2. **Checkout & SCP:**  The runner pulls Repo B and uses appleboy/scp-action to copy docker-compose.yml and configs to the on-prem VM.  
3. **Remote Execution:**  The runner SSHs into the box to run docker compose pull and docker compose up \-d.While automation handles the "happy path," we must enforce strict governance for failures and regressions.

#### 6\. Safety and Governance: Rollbacks and Tagging Strategies

In GitOps,  **manual server changes are forbidden** . If a tech lead manually edits a ConfigMap on the cluster, the GitOps controller will immediately overwrite it to match the state in Git.

##### The Golden Rule: Backporting Fixes

The biggest risk in concurrent releases is  **regression** . If a bug is found in Prod (v1.1) while QA is testing v1.2, you must:

1. **Fix:**  Branch hotfix/v1.1.1 from release/v1.1 in Repo A.  
2. **Deploy:**  Build v1.1.1, update Repo B prod/ to this tag.  
3. **Cherry-pick Forward:**  Immediately cherry-pick that fix into release/v1.2 and main so the bug doesn't reappear in the next release.

##### Rollback Strategy Comparison

Method,When to Use,"Risk of ""Drift"""  
Git Revert,Standard bugs / minor issues.,None.  Git remains the source of truth.  
Argo CD Panic Button,Critical outages (CrashLoops).,High.  The server moves to v1 while Git says v2.

##### Environment-Prefixed Tagging & Audit

Repo B uses tags like prod-v1.2.0 to create immutable snapshots. This allows for  **Batch Diffs**  to audit changes across 50+ files simultaneously:

* **Command:**  git diff prod-v1.1.0 prod-v1.2.0 This instantly reveals every environment variable and image tag change between two specific releases.

##### Summary: The 4-Step End-to-End Release Flow

1. **Build & Push:**  Repo A CI builds the Java app and pushes v1.2.0 to JFrog.  
2. **Automated PR:**  Repo A CI automatically creates a PR in Repo B, updating the image tag in deploy/prod/.  
3. **Merge & Sync:**  Tech Leads approve the PR; Argo CD or GitHub Actions syncs the state.  
4. **Tag Snapshot:**  Repo B automation applies a prod-v1.2.0 tag to the commit for future audit and rollback capability.

