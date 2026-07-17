### Standard Operating Procedure: GitOps-Driven Java Application Deployment

#### 1\. Governance and Architectural Philosophy

This Standard Operating Procedure (SOP) establishes a rigorous framework for the separation of application state from execution binaries, a core requirement for high-availability distributed systems. By decoupling configuration from code, we institutionalize the "12-Factor App" methodology, ensuring that deployment artifacts remain immutable. We build exactly one environment-agnostic Docker image per release; this binary is promoted across environments without modification, guaranteeing that the exact code verified in QA is what executes in Production. This architectural sovereignty eliminates "it works on my machine" variances and ensures that environmental specificity is injected only at runtime.To enforce clear ownership boundaries and operational stability, we mandate a  **Two-Repository Strategy** . This separation prevents configuration drift from triggering unnecessary CI/CD churn or invalidating build caches.| Feature | Repo A: Application Source | Repo B: GitOps Configuration || \------ | \------ | \------ || **Contents** | Java source code, pom.xml, and Dockerfile. | Declarative manifests (K8s YAML, Docker Compose) and config files (application.yml). || **Purpose** | Compilation of the immutable binary artifact. | Management of the global environment state and reconciliation. || **Access Controls** | Engineering teams (Feature development). | Tech Leads and DevOps (Environment governance). || **Storage Logic** | JFrog Artifactory (Binary blobs). | Git (Text-based version control). |  
A critical architectural decision is the storage of configurations in Git rather than a binary repository like JFrog. JFrog treats files as "opaque blobs," making structural comparisons and lineage tracking impossible at the Pull Request (PR) stage. By utilizing Git, every adjustment—from database connection pool tuning to memory limit constraints—is captured as a text-based commit. This provides a transparent, permanent audit trail and enables sophisticated text-based diffs, allowing architects to validate infrastructure changes before they are reconciled against live environments.

#### 2\. Local Development and Environment Isolation

To maximize developer velocity, local development configurations must maintain environmental sovereignty, remaining entirely decoupled from the GitOps repository. This isolation ensures that developers can iterate autonomously without risking interference with shared infrastructure or being delayed by the governance protocols required for deployed environments.

##### Repo A Configuration Protocol

The Application Repository (Repo A) must be self-contained for local execution.

* **Target:**  src/main/resources/application-local.yml  
* **Constraint:**  This file is reserved for local profile overrides (e.g., localhost pointers, mock service URLs).  **Commitment of production secrets to this file is a Tier-1 security violation.**

##### Developer Readiness Checklist

Prior to code submission, engineers must validate their features within a self-contained local context:

*   **Dependency Abstraction:**  Utilize  **Testcontainers**  for integration testing or a root-level docker-compose-local.yml for local service orchestration.  
*   **Profile Activation:**  Configure the IDE to target the local Spring profile (e.g., \-Dspring.profiles.active=local).  
*   **Agnotic Validation:**  Ensure the feature functions against local mocks before attempting integration with the broader pipeline.

##### Configuration Injection Lifecycle

When a feature introduces a new configuration requirement (e.g., a new environment variable), the following protocol is mandatory:

1. **Local Update (Repo A):**  Update application-local.yml and verify the feature locally.  
2. **GitOps Update (Repo B):**  Submit a PR to Repo B to introduce the new variable into the target environment directories (e.g., deploy/qa/).This sequence ensures that local validation is a prerequisite for any modification to the organization's managed environment state.

#### 3\. The GitOps Release Lifecycle and Promotion Workflow

This SOP mandates a transition from "Branch-per-Environment" to a  **"Directory-per-Environment"**  model on a single main branch in Repo B. This move eliminates "Merge Hell" and ensures that the main branch serves as the unified "Single Source of Truth."

##### Release Flow Branching Strategy (Repo A)

The application source follows a  **Release Flow**  protocol to manage the lifecycle of the Java binary:

* **main (The Trunk):**  The stable integration point for all feature branches.  
* **feature/\*:**  Short-lived branches for development, integrated into main via PR.  
* **release/\*:**  Created from main for release candidates (e.g., release/v1.2).  
* **The Golden Rule (Cherry-pick Forward):**  Any bug fix identified during QA must be applied to the release/\* branch and immediately  **cherry-picked back to**  **main** . This prevents regressions in subsequent release cycles.  
* **hotfix/\*:**  Emergency branches originating from the active release branch to address production incidents.

##### The Promotion Mechanism

Promotion is achieved via declarative file updates in Repo B, rather than traditional Git merges. Moving a release from QA to Production involves updating the deploy/prod/ directory in Repo B to match the image tag and configuration verified in deploy/qa/.This directory-based model is architecturally superior as it prevents environment drift. In merge-based systems, QA-specific overrides (like mock credentials) often accidentally leak into Production during a merge. In our model, each environment is explicitly isolated in its own directory, ensuring that only intended updates—specifically the verified binary version—move forward through the reconciliation loop.

#### 4\. Environment-Specific Deployment Protocols

Our GitOps framework maintains a unified trigger mechanism while supporting heterogeneous infrastructure targets through specialized injection points.

##### EKS (Argo CD) Workflow

Argo CD acts as the GitOps controller, monitoring the deploy/k8s-eks/overlays/ path.

* **Kustomize Overlays:**  We utilize Kustomize to manage manifests. The base/ directory contains core logic, while overlays for QA and Prod provide environment-specific overrides.  
* **Injection Point:**  Environmental state is injected via  **Kubernetes ConfigMaps** .  
* **Secrets Management:**  Secrets are managed via the  **External Secrets Operator (ESO)** , which pulls sensitive data from the enterprise vault into the cluster at runtime.  
* **Protocol (Stakater Reloader):**  To ensure high availability, the  **Stakater Reloader**  operator is mandatory. It detects ConfigMap/Secret updates and triggers a rolling restart of Java pods to ensure the new state is applied without manual intervention.

##### On-Prem (GitHub Actions) Workflow

For non-Kubernetes environments, GitHub Actions simulates GitOps behavior for Docker Compose targets:

1. **Filter Triggers:**  The workflow triggers exclusively on pushes to deploy/on-prem/prod/\*\*.  
2. **Secrets Injection:**  Credentials are pulled from  **GitHub Secrets**  during the workflow execution.  
3. **Handoff (SCP):**  The appleboy/scp-action transfers the docker-compose.yml and environment-specific configs to the target VM.  
4. **Reconciliation (SSH):**  The appleboy/ssh-action executes docker compose pull and docker compose up \-d.  
5. **Injection Point:**  Configuration is applied via  **Docker Compose volume mounts** , mapping the synchronized Git directories to the container’s internal file system.

##### Automated Handoff

The link between Repo A and Repo B is automated. Upon a successful build and binary push to JFrog, Repo A's CI pipeline automatically generates a PR in Repo B to update the relevant image tag. This ensures a seamless transition from binary creation to environmental declaration.

#### 5\. Multi-Release Management and Rollback Strategies

As release cycles overlap, managing concurrent deployments requires a sophisticated directory strategy to prevent environment collision.

##### Ephemeral Environments

When multiple releases (e.g., v1.2 and v1.3) require QA simultaneously, we utilize  **Ephemeral Environments** . Using Argo CD and Kustomize, we provision temporary directories (e.g., deploy/k8s-eks/overlays/qa-v1.2/) that target dedicated Kubernetes namespaces. This allows for parallel validation without overwriting the primary QA state. Once UAT is complete, the directory is deleted, and Argo CD automatically tears down the associated resources.

##### Rollback Comparison Matrix

Operational recovery follows two distinct protocols:| Feature | Method 1: Git Revert (Standard) | Method 2: Argo CD Manual Override (Emergency) || \------ | \------ | \------ || **Speed** | 2–5 minutes. | \< 30 seconds. || **Mechanism** | New commit in Repo B. | Direct UI "Rollback" action. || **Audit Trail** | Comprehensive; preserves history. | Temporary drift; breaks GitOps alignment. || **Status** | Preferred / Safe. | Emergency Only. |

##### The "Panic Button" and Drift Correction

The Argo CD manual rollback is an emergency measure that triggers a  **"Yellow Warning / OutOfSync"**  status. This indicates that the cluster state has diverged from the declarative state in Git.  **Mandatory Corrective Action:**  Immediately following an emergency rollback, a git revert must be merged into Repo B to align the Git repository with the cluster’s current version, returning the system to a "Synced" state.

#### 6\. Snapshot Management and Auditability

To manage the complexity of 50+ configuration files, Repo B functions as a permanent historical ledger through environment-prefixed tagging.

##### Tagging Nomenclature

We utilize immutable "bookmarks" to capture the state of the infrastructure:

* qa-v1.3.0-rc1: Snapshot of the QA environment for a specific release candidate.  
* prod-v1.2.0: The definitive state of the Production environment at release time.

##### Forensic Auditing via "Batch Diff"

These tags enable engineers to perform forensic analysis of the entire infrastructure state. By executing a  **Batch Diff** , every configuration change and image tag update between two production states can be reviewed in a single view:  
\# Evaluate every delta between two production deployments  
git diff prod-v1.1.0 prod-v1.2.0

This command allows for the rapid detection of unauthorized environment variable drift or unintended resource limit changes. By adhering to these protocols, the organization ensures that the GitOps repository is not merely a deployment tool, but a high-fidelity, auditable ledger of the entire enterprise infrastructure.  
