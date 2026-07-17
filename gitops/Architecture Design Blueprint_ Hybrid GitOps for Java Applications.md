### Architecture Design Blueprint: Hybrid GitOps for Java Applications

#### 1\. Architectural Philosophy: The 12-Factor Strategy

Strategic excellence in modern systems engineering requires a fundamental separation between application logic and environmental state. By aligning with the "12-Factor App" methodology, we decouple the "what" (the application) from the "where" (the environment). This separation ensures that Java applications remain portable and resilient across diverse infrastructure—from managed Amazon EKS clusters to legacy on-premises virtual machines—eliminating the variability that often plagues hybrid deployments.

##### Core Principles: The Immutable Binary

At the heart of this strategy is the concept of the  **Immutable Binary** . We mandate building exactly one Docker image per release. This identical image—verified by its unique digest—is promoted through QA, UAT, and eventually Production without being rebuilt or repackaged. This principle ensures environmental parity: if the binary was verified in a staging environment, it is guaranteed to behave identically in production, as the execution logic remains frozen.

##### The "So What?" Layer: Verification Integrity

Externalizing configuration into environment variables and volume-mounted files (ConfigMaps or .env files) guarantees that the exact code verified during high-fidelity QA testing is what reaches the end user. This methodology eliminates the risk of "build-time" bugs where environment-specific injection during a Maven phase or Docker build might introduce regressions that were never observed in earlier lifecycle stages.

#### 2\. The Two-Repository Strategic Framework

Industry standards for professional GitOps necessitate a dual-repository model. This separation resolves the paradox of misaligned branch states where application code moves at the speed of development while infrastructure state must remain governed and stable.

##### Comparative Analysis

Feature,Repo A: Application Repository,Repo B: GitOps Deployment Repository  
Primary Contents,"Java source code, pom.xml, Dockerfile","K8s Manifests, Kustomize overlays, .env files"  
Branching Model,Release Flow (Trunk-based variation),Single long-lived main branch  
Primary Purpose,Artifact creation (The Binary Engine),State management (The Single Source of Truth)  
Triggered Actions,"CI builds, image push to JFrog, Hand-off PR","Argo CD syncs, GitHub Action SSH deployments"

##### The "So What?" Layer: Velocity and Security

The primary strategic driver for repo separation is the prevention of  **"Endless CI Loops."**  In a combined repository, a minor adjustment to a database timeout in a production config file would trigger a 15-minute Maven build and a redundant Docker image push. Separation ensures configuration updates are near-instant. Furthermore, it provides granular security; while 50 developers may have access to Repo A, write access to Repo B’s production directories can be restricted to senior DevOps architects, protecting the environment's integrity.

#### 3\. Repo A: Application Lifecycle & Artifact Management

Repo A functions as the "Binary Engine," transforming source code into environment-agnostic artifacts. We utilize  **JFrog Artifactory**  as the absolute source of truth for all compiled assets, effectively acting as the bridge between the two repositories.

##### Java-Specific Directory Structure

The application repository maintains a self-contained structure to maximize developer velocity:  
├── src/                    \# Java source code  
├── pom.xml                 \# Maven dependencies  
├── Dockerfile              \# Environment-agnostic container build  
├── docker-compose-local.yml\# Local dependencies (DBs, Brokers)  
└── src/main/resources/  
    ├── application.yml     \# Base default properties  
    └── application-local.yml \# Overrides for local IntelliJ execution

##### JFrog Integration: Auditability vs. Immutability

JFrog Artifactory stores "heavy" assets (Docker images and .jar files) that would otherwise bloat Git.

* **Auditability:**  JFrog provides binary vulnerability scanning and metadata tracking that Git cannot offer.  
* **Immutability:**  Once an image is pushed to JFrog, it is never modified. Version control for text-diffs remains in Git, while the binary remains locked.

##### The "So What?" Layer: Local Developer Autonomy

By including application-local.yml and  **Testcontainers**  (or docker-compose-local.yml), developers can execute the entire application stack within IntelliJ. They stay productive by pointing to local mock services and databases without needing to interact with Repo B, ensuring that local development remains fast, isolated, and autonomous.

#### 4\. Repo B: The GitOps "Directory-Per-Environment" Structure

Repo B manages the infrastructure state using a "Directory-per-Environment" model on a single main branch. This approach is superior to "Branch-per-Environment" as it eliminates merge hell and prevents "hotfixes" from diverging across long-lived branches.

##### Master Directory Tree

The deploy/ root is organized to support hybrid targets with precision:  
deploy/  
├── k8s-eks/                \# Kubernetes targets (Argo CD)  
│   ├── base/               \# Common deployments and services  
│   └── overlays/  
│       ├── qa/             \# Kustomization.yaml, ConfigMaps, image tags  
│       └── prod/           \# Production-specific manifests  
└── on-prem/                \# VM targets (GitHub Actions)  
    ├── docker-compose.yml  \# Shared compose definition  
    ├── qa/  
    │   ├── application.yml \# QA properties  
    │   └── .env            \# QA Compose variables  
    └── prod/  
        ├── application.yml \# Production properties  
        └── .env            \# Production Compose variables

##### State Management: Kustomize and ESO

We utilize  **Kustomize**  to manage "base" manifests with environment-specific "overlays." For secret management in EKS, we integrate  **External Secrets Operator (ESO)** , which fetches sensitive data from external vaults and injects them as Kubernetes Secrets, ensuring no plain-text passwords reside in Git.

##### The "So What?" Layer: Auditability and Snapshots

The main branch serves as the unified audit log. Stakeholders can use  **environment-prefixed tags**  (e.g., prod-v1.2.0) to create immutable snapshots. This allows for  **"batch diffs"** —executing git diff prod-v1.1.0 prod-v1.2.0 provides an instantaneous audit of every configuration change and image tag update across 50+ files in a single view.

#### 5\. Deployment Orchestration: EKS vs. On-Premises

Synchronizing a single source of truth to a hybrid compute environment requires two distinct orchestration models.

##### EKS Workflow (Pull-Based via Argo CD)

Argo CD monitors the k8s-eks/ directories.

1. **Drift Detection:**  Argo CD identifies mismatches between Git and the EKS cluster.  
2. **Automated Sync:**  Changes are applied to the cluster.  
3. **Pod Refresh:**  We utilize  **Stakater Reloader**  to watch ConfigMaps and Secrets; it automatically triggers a rolling restart of pods so Java applications pick up new configurations without manual intervention.

##### On-Premises Workflow (Push-Based via GitHub Actions)

Since VMs lack native GitOps operators, we simulate the behavior using the following push-based sequence:

1. **Filter Triggers:**  The action runs only on changes to deploy/on-prem/\*\*.  
2. **Secure Copy:**  Using appleboy/scp-action, the workflow transfers docker-compose.yml and the environment-specific directory to the target VM.  
3. **Remote Execution:**  Using appleboy/ssh-action, the workflow executes docker compose pull && docker compose up \-d to reconcile the running containers with the new Git state.

##### The "So What?" Layer: Operational Continuity

The "Pull" model (Argo CD) provides continuous reconciliation and drift protection for Kubernetes. The "Push" model (GHA) extends GitOps-like governance to legacy on-premises environments, ensuring that even bare-metal deployments benefit from the same audit trails and version control as the cloud.

#### 6\. Release Management and Promotion Flow

We employ  **Release Flow** , a trunk-based variation that manages concurrent releases across the lifecycle.

##### The Promotion Sequence and Handoff

1. **Development:**  Features are merged into Repo A's main branch.  
2. **Release Cut:**  A release/v1.x branch is created in Repo A, triggering a build and image push to JFrog (e.g., myapp:1.2.0).  
3. **The Automated Handoff:**  A CI step in Repo A automatically clones Repo B and opens a Pull Request to update the image tag in the qa/ directory.  
4. **Promotion:**  After QA verification, a PR in Repo B moves the image tag and configurations to the prod/ directory.

##### The "So What?" Layer: The Backporting Mandate

To prevent regressions in concurrent releases, any fix applied to a release branch (e.g., release/v1.1)  **must be cherry-picked into all other active release branches (v1.2, v1.3, etc.) AND the**  **main**  **branch.**  This ensures that a production fix is permanently integrated into the future roadmap.

#### 7\. Operational Resilience: Rollback and Recovery

In GitOps, the desired state is defined by Git. "Fixing the server" means reconciling the Git state.

##### Dual-Method Rollback Strategy

1. **Method 1: Git Revert (Standard):**  Revert the bad merge in Repo B. This creates a new commit, maintaining a perfect audit trail of why and when the rollback occurred.  
2. **Method 2: The Panic Button (Emergency):**  Use the Argo CD "History and Rollback" feature to revert the cluster state in seconds.

##### The "So What?" Layer: Managing Drift

Using the "Panic Button" causes an  **"OutOfSync" Yellow Warning**  in Argo CD, as the cluster no longer matches Repo B. Argo CD suspends auto-syncing to prevent the broken version from being redeployed.  **Operational mandate:**  Every emergency rollback must be followed by a "Git Revert" or "Fix-Forward" in Repo B to reconcile the source of truth with the running environment and restore the green sync state.**Final Summary:**  This two-repo, directory-based architecture provides a professional, scalable, and immutable deployment lifecycle. By separating the binary engine from the configuration state, Java applications achieve unprecedented reliability across EKS and on-premises environments.\# Architecture Design Blueprint: Hybrid GitOps for Java Applications

#### 1\. Architectural Philosophy: The 12-Factor Strategy

Strategic excellence in modern systems engineering requires a fundamental separation between application logic and environmental state. By aligning with the "12-Factor App" methodology, we decouple the "what" (the application) from the "where" (the environment). This separation ensures that Java applications remain portable and resilient across diverse infrastructure—from managed Amazon EKS clusters to legacy on-premises virtual machines—eliminating the variability that often plagues hybrid deployments.

##### Core Principles: The Immutable Binary

At the heart of this strategy is the concept of the  **Immutable Binary** . We mandate building exactly one Docker image per release. This identical image—verified by its unique digest—is promoted through QA, UAT, and eventually Production without being rebuilt or repackaged. This principle ensures environmental parity: if the binary was verified in a staging environment, it is guaranteed to behave identically in production, as the execution logic remains frozen.

##### The "So What?" Layer: Verification Integrity

Externalizing configuration into environment variables and volume-mounted files (ConfigMaps or .env files) guarantees that the exact code verified during high-fidelity QA testing is what reaches the end user. This methodology eliminates the risk of "build-time" bugs where environment-specific injection during a Maven phase or Docker build might introduce regressions that were never observed in earlier lifecycle stages.

#### 2\. The Two-Repository Strategic Framework

Industry standards for professional GitOps necessitate a dual-repository model. This separation resolves the paradox of misaligned branch states where application code moves at the speed of development while infrastructure state must remain governed and stable.

##### Comparative Analysis

Feature,Repo A: Application Repository,Repo B: GitOps Deployment Repository  
Primary Contents,"Java source code, pom.xml, Dockerfile","K8s Manifests, Kustomize overlays, .env files"  
Branching Model,Release Flow (Trunk-based variation),Single long-lived main branch  
Primary Purpose,Artifact creation (The Binary Engine),State management (The Single Source of Truth)  
Triggered Actions,"CI builds, image push to JFrog, Hand-off PR","Argo CD syncs, GitHub Action SSH deployments"

##### The "So What?" Layer: Velocity and Security

The primary strategic driver for repo separation is the prevention of  **"Endless CI Loops."**  In a combined repository, a minor adjustment to a database timeout in a production config file would trigger a 15-minute Maven build and a redundant Docker image push. Separation ensures configuration updates are near-instant. Furthermore, it provides granular security; while 50 developers may have access to Repo A, write access to Repo B’s production directories can be restricted to senior DevOps architects, protecting the environment's integrity.

#### 3\. Repo A: Application Lifecycle & Artifact Management

Repo A functions as the "Binary Engine," transforming source code into environment-agnostic artifacts. We utilize  **JFrog Artifactory**  as the absolute source of truth for all compiled assets, effectively acting as the bridge between the two repositories.

##### Java-Specific Directory Structure

The application repository maintains a self-contained structure to maximize developer velocity:  
├── src/                    \# Java source code  
├── pom.xml                 \# Maven dependencies  
├── Dockerfile              \# Environment-agnostic container build  
├── docker-compose-local.yml\# Local dependencies (DBs, Brokers)  
└── src/main/resources/  
    ├── application.yml     \# Base default properties  
    └── application-local.yml \# Overrides for local IntelliJ execution

##### JFrog Integration: Auditability vs. Immutability

JFrog Artifactory stores "heavy" assets (Docker images and .jar files) that would otherwise bloat Git.

* **Auditability:**  JFrog provides binary vulnerability scanning and metadata tracking that Git cannot offer.  
* **Immutability:**  Once an image is pushed to JFrog, it is never modified. Version control for text-diffs remains in Git, while the binary remains locked.

##### The "So What?" Layer: Local Developer Autonomy

By including application-local.yml and  **Testcontainers**  (or docker-compose-local.yml), developers can execute the entire application stack within IntelliJ. They stay productive by pointing to local mock services and databases without needing to interact with Repo B, ensuring that local development remains fast, isolated, and autonomous.

#### 4\. Repo B: The GitOps "Directory-Per-Environment" Structure

Repo B manages the infrastructure state using a "Directory-per-Environment" model on a single main branch. This approach is superior to "Branch-per-Environment" as it eliminates merge hell and prevents "hotfixes" from diverging across long-lived branches.

##### Master Directory Tree

The deploy/ root is organized to support hybrid targets with precision:  
deploy/  
├── k8s-eks/                \# Kubernetes targets (Argo CD)  
│   ├── base/               \# Common deployments and services  
│   └── overlays/  
│       ├── qa/             \# Kustomization.yaml, ConfigMaps, image tags  
│       └── prod/           \# Production-specific manifests  
└── on-prem/                \# VM targets (GitHub Actions)  
    ├── docker-compose.yml  \# Shared compose definition  
    ├── qa/  
    │   ├── application.yml \# QA properties  
    │   └── .env            \# QA Compose variables  
    └── prod/  
        ├── application.yml \# Production properties  
        └── .env            \# Production Compose variables

##### State Management: Kustomize and ESO

We utilize  **Kustomize**  to manage "base" manifests with environment-specific "overlays." For secret management in EKS, we integrate  **External Secrets Operator (ESO)** , which fetches sensitive data from external vaults and injects them as Kubernetes Secrets, ensuring no plain-text passwords reside in Git.

##### The "So What?" Layer: Auditability and Snapshots

The main branch serves as the unified audit log. Stakeholders can use  **environment-prefixed tags**  (e.g., prod-v1.2.0) to create immutable snapshots. This allows for  **"batch diffs"** —executing git diff prod-v1.1.0 prod-v1.2.0 provides an instantaneous audit of every configuration change and image tag update across 50+ files in a single view.

#### 5\. Deployment Orchestration: EKS vs. On-Premises

Synchronizing a single source of truth to a hybrid compute environment requires two distinct orchestration models.

##### EKS Workflow (Pull-Based via Argo CD)

Argo CD monitors the k8s-eks/ directories.

1. **Drift Detection:**  Argo CD identifies mismatches between Git and the EKS cluster.  
2. **Automated Sync:**  Changes are applied to the cluster.  
3. **Pod Refresh:**  We utilize  **Stakater Reloader**  to watch ConfigMaps and Secrets; it automatically triggers a rolling restart of pods so Java applications pick up new configurations without manual intervention.

##### On-Premises Workflow (Push-Based via GitHub Actions)

Since VMs lack native GitOps operators, we simulate the behavior using the following push-based sequence:

1. **Filter Triggers:**  The action runs only on changes to deploy/on-prem/\*\*.  
2. **Secure Copy:**  Using appleboy/scp-action, the workflow transfers docker-compose.yml and the environment-specific directory to the target VM.  
3. **Remote Execution:**  Using appleboy/ssh-action, the workflow executes docker compose pull && docker compose up \-d to reconcile the running containers with the new Git state.

##### The "So What?" Layer: Operational Continuity

The "Pull" model (Argo CD) provides continuous reconciliation and drift protection for Kubernetes. The "Push" model (GHA) extends GitOps-like governance to legacy on-premises environments, ensuring that even bare-metal deployments benefit from the same audit trails and version control as the cloud.

#### 6\. Release Management and Promotion Flow

We employ  **Release Flow** , a trunk-based variation that manages concurrent releases across the lifecycle.

##### The Promotion Sequence and Handoff

1. **Development:**  Features are merged into Repo A's main branch.  
2. **Release Cut:**  A release/v1.x branch is created in Repo A, triggering a build and image push to JFrog (e.g., myapp:1.2.0).  
3. **The Automated Handoff:**  A CI step in Repo A automatically clones Repo B and opens a Pull Request to update the image tag in the qa/ directory.  
4. **Promotion:**  After QA verification, a PR in Repo B moves the image tag and configurations to the prod/ directory.

##### The "So What?" Layer: The Backporting Mandate

To prevent regressions in concurrent releases, any fix applied to a release branch (e.g., release/v1.1)  **must be cherry-picked into all other active release branches (v1.2, v1.3, etc.) AND the**  **main**  **branch.**  This ensures that a production fix is permanently integrated into the future roadmap.

#### 7\. Operational Resilience: Rollback and Recovery

In GitOps, the desired state is defined by Git. "Fixing the server" means reconciling the Git state.

##### Dual-Method Rollback Strategy

1. **Method 1: Git Revert (Standard):**  Revert the bad merge in Repo B. This creates a new commit, maintaining a perfect audit trail of why and when the rollback occurred.  
2. **Method 2: The Panic Button (Emergency):**  Use the Argo CD "History and Rollback" feature to revert the cluster state in seconds.

##### The "So What?" Layer: Managing Drift

Using the "Panic Button" causes an  **"OutOfSync" Yellow Warning**  in Argo CD, as the cluster no longer matches Repo B. Argo CD suspends auto-syncing to prevent the broken version from being redeployed.  **Operational mandate:**  Every emergency rollback must be followed by a "Git Revert" or "Fix-Forward" in Repo B to reconcile the source of truth with the running environment and restore the green sync state.**Final Summary:**  This two-repo, directory-based architecture provides a professional, scalable, and immutable deployment lifecycle. By separating the binary engine from the configuration state, Java applications achieve unprecedented reliability across EKS and on-premises environments.  
