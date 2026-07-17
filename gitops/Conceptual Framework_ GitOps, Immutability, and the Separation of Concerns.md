### Conceptual Framework: GitOps, Immutability, and the Separation of Concerns

#### 1\. The Core Philosophy: Separating Code from State

In modern enterprise architecture, the  **12-Factor App**  methodology is not a suggestion—it is a requirement for survival. At its core, this methodology demands a total separation of  **Code**  (binaries and logic) from  **State**  (environment-specific configuration).Code comprises the compiled Java instructions, shared libraries, and application logic that remain constant. State involves database credentials, API endpoints, and resource limits that vary across environments. By stripping configuration out of your delivery artifacts, you create  **environment-agnostic Docker images** . This ensures that the exact binary verified by your QA team is the same binary running in Production, eliminating "configuration drift" and significantly accelerating Time-to-Market while ensuring compliance for frameworks like SOC2.**Key Principle: Build Once, Promote Everywhere**   **You must build exactly one immutable Docker image per release and promote that identical artifact through every stage of your pipeline (QA, UAT, Prod, On-Prem). The artifact remains a constant; only the environment-specific configuration surrounding it changes.**Establishing this separation of concerns is the first step; the second is determining the physical storage strategy for these distinct assets.

#### 2\. The Two Pillars of Storage: JFrog vs. Git

To maintain a high-integrity pipeline, we must distinguish between "heavy" binaries and "text-based" declarations. JFrog Artifactory is built for the former, while Git is built for the latter.| Asset Type | Storage Location (Source of Truth) | Primary Reason || \------ | \------ | \------ || **Environment Configs** | **Git** | Text-based; requires version control, granular diffs, and Pull Request (PR) reviews. || **Deployment Manifests** | **Git** | Infrastructure as Code (IaC); must be versioned alongside the configurations they apply to. || **Docker Images** | **JFrog Artifactory** | Heavy binary blobs; optimized for vulnerability scanning and high-speed distribution to clusters. || **Compiled Code (.jar/.war)** | **JFrog Artifactory** | Raw Java artifacts used for dependency management prior to image wrapping. |

##### Why Configuration Never Belongs in JFrog

Storing text-based configuration inside a binary manager is a structural failure for three specific reasons:

1. **Auditability:**  JFrog cannot handle text-based "diffs." Git provides a permanent record of  *who*  changed a timeout or database URL and  *why* , linked directly to a PR.  
2. **Immutability:**  A Docker image is immutable. If configuration is bundled inside, changing a single environment variable requires a full rebuild of the image, breaking the "Build Once" rule.  
3. **Single Source of Truth:**  Git acts as the definitive ledger of "how the environment should behave," while JFrog is the registry of "the code that executes."Storing these assets correctly is only half the battle; we must now organize the Git repository to prevent environment divergence.

#### 3\. Structural Strategy: Directory-Per-Environment vs. Branch-Per-Environment

Mapping Git branches to environments (e.g., a qa branch and a prod branch) is a dangerous anti-pattern and a technical debt accelerator. This approach leads to  **Merge Hell** , where hotfixes applied to Production are never merged back to QA, causing the environments to drift until they are no longer comparable.The industry standard is the  **Directory-Per-Environment**  model on a single main branch.

* **Branch-Per-Environment (Anti-Pattern):**  High risk of divergence; merge conflicts during promotion; zero visibility into the total infrastructure state.  
* **Directory-Per-Environment (Recommended):**  Unified view of all environments; Kustomize/Helm bases enforce consistency; natively supported by GitOps engines like Argo CD.

##### Recommended Directory Structure

deploy/                     
├── k8s-eks/              \# Target for Argo CD (Pull Model)  
│   ├── base/             \# Common Kubernetes manifests  
│   └── overlays/  
│       ├── qa/           \# QA ConfigMaps, Secrets, image tags  
│       └── prod/         \# Prod ConfigMaps, Secrets, image tags  
└── on-prem/              \# Target for GitHub Actions (Push Model)  
    ├── docker-compose.yml  
    ├── qa/               \# application.yml and .env for QA  
    └── prod/             \# application.yml and .env for Prod

##### The Promotion Workflow

Promoting a change from QA to Production in this model is a transparent, 3-step process:

1. **Update QA:**  Modify the configuration in deploy/k8s-eks/overlays/qa/ and merge into main.  
2. **Verify:**  Confirm the system behaves as expected in the QA environment.  
3. **Promote to Prod:**  Open a new PR to copy the verified configuration (or image tag) into the deploy/prod/ directory.This directory structure eventually necessitates a physical split between your source code and your configuration to avoid versioning paradoxes.

#### 4\. The Two-Repository Architecture

In a professional GitOps ecosystem, you must implement the  **Two-Repo Strategy**  to separate the "conveyor belt" from the "ledger."

* **Repo A (Application Repository):**  Contains Java source, pom.xml, and Dockerfile. Its sole purpose is building the artifact and pushing it to JFrog.  
* **Repo B (GitOps Repository):**  Contains only deployment manifests and environment configs. This is the source of truth for your running state.

##### Headaches Solved by Separation

* **Version Mismatch:**  Prevents the "bleeding edge" code on Repo A's main branch from being confused with the stable configuration sitting in Repo B.  
* **CI Loops:**  Prevents a simple configuration tweak (like a password change) from triggering a 20-minute Maven build and Docker push.  
* **Security/Access Control:**  Allows broad developer access to Repo A, while restricting Repo B (specifically the prod/ folders) to Tech Leads or DevOps.  
* **Audit Trails:**  Repo B becomes a clean, noise-free history of infrastructure movements.

##### The Developer Experience and Automation

Developers remain autonomous within Repo A by using an application-local.yml and  **Testcontainers** . This allows them to run the full app and its dependencies (Postgres, Kafka) locally in IntelliJ without ever touching the GitOps repo.When code is merged in Repo A, the CI pipeline should  **automatically**  create a PR in Repo B to update the image tag. This ensures the handoff is machine-driven, while the final promotion remains a human-governed review.Moving from architecture to operations, we must define how to handle the inevitable need to revert changes.

#### 5\. Reliability and Recovery: Rollbacks and Snapshots

In a GitOps world, the server is a mirror of Git. If you manually change a setting on a server, a "Pull" tool like Argo CD will detect the drift and overwrite it. Therefore, all recovery actions must originate in Git.

##### The Git Revert Workflow (Standard Practice)

To roll back a binary from v2 to v1, follow these steps:

1. Identify the specific Git commit in Repo B that performed the v1 to v2 upgrade.  
2. Execute a git revert on that commit to create a "reversal" commit.  
3. Merge the revert into the main branch.  
4. **For EKS:**  Argo CD detects the change and pulls the v1 configuration. Use  **Stakater Reloader**  to ensure Kubernetes pods automatically perform a rolling restart to pick up the revised ConfigMap.  
5. **For On-Prem:**  A GitHub Action triggers on the merge, using  **appleboy/scp-action**  to copy the v1 files and  **appleboy/ssh-action**  to run docker compose up \-d.**⚠️ Warning: The Panic Button and Drift**  Argo CD’s "History and Rollback" UI is an emergency tool. Using it creates an  **"OutOfSync" (Yellow Warning)**  state because the cluster no longer matches Git. You must follow up with a git revert in Repo B to reconcile the state and resume auto-syncing.

##### Tagging Strategy for Snapshots

For systems with complex configurations, use  **Environment-Prefixed Tags**  in Repo B (e.g., prod-v1.2.0). This creates an immutable snapshot of all 50+ config files at the moment of release. This allows for "batch diffs" (e.g., git diff prod-v1.1.0 prod-v1.2.0), providing a total overview of every variable change and image update that occurred between production deployments.By strictly separating code from state and managing your infrastructure via Git, you ensure a deployment pipeline that is stable, auditable, and prepared for rapid recovery.  
