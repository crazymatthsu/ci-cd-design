# **GitOps for Java App Deployment**

[https://gemini.google.com/app/7cd096e0859cd1d2](https://gemini.google.com/app/7cd096e0859cd1d2)

*User prompt: So question about the CICD I have a Java application and then I have a configuration for different environment and that I want to be a talker and managed and to be able to run the stalker image in the QAM pro also on prime and also in EKS and I want to using the same darker image, but I want to have different configuration file in different environment and I want to use get ups like Argo CD to sync a configuration from gate to config map and then for on prime, how can I achieve this using GitHub GitHub action similar this get ups like behavior I change config and get you auto sync for me using a hip-hop action to a Orem box and should I bundle the configuration in the darker image or should I using the same image across all the environment, but in cooperative using config map in on prime boxes, using the dark composed to mount the config directory, can you help me to design the kit directory structure and then how can I achieve this get ups action kind of design principle I change convey good and then it will help me to auto sync to either on prime machine or to Cooper*

Response: This is a classic enterprise architecture challenge, and your instinct to separate the configuration from the deployment artifact is spot on.

To answer your first question directly: **Do not bundle the configuration in the Docker image.** You should build exactly one immutable Docker image per release and promote that identical image through QA, Prod, On-Prem, and EKS.

This perfectly aligns with the "12-Factor App" methodology. By keeping the image environment-agnostic, you guarantee that the exact code tested in QA is what runs in Production. You will achieve the environment specificity using Kubernetes ConfigMaps in EKS and Docker Compose volume mounts for your on-prem boxes.

Here is how you can design the directory structure and the GitOps workflows to achieve this seamlessly.

## **1\. The Git Directory Structure**

For a clean GitOps workflow, it is highly recommended to separate your application source code from your deployment manifests. This prevents a simple configuration change from triggering a full Java/Maven build and Docker image push.

If you are using a single repository (or a dedicated deployment repository), structure it like this:

`├── app/                      # Your Java Application`  
`│   ├── src/`  
`│   ├── pom.xml`  
`│   └── Dockerfile            # Builds the environment-agnostic image`  
`│`  
`└── deploy/                   # The GitOps Configuration Root`  
    `│`  
    `├── k8s-eks/              # Target for Argo CD`  
    `│   ├── base/             # Base deployments, services (Kustomize/Helm)`  
    `│   ├── overlays/`  
    `│   │   ├── qa/           # QA ConfigMaps, Secrets, specific image tags`  
    `│   │   └── prod/         # Prod ConfigMaps, Secrets`  
    `│`  
    `└── on-prem/              # Target for GitHub Actions`  
        `├── docker-compose.yml # Base compose file referencing your registry image`  
        `├── qa/`  
        `│   ├── application.yml    # QA specific Java properties`  
        `│   └── .env               # Environment variables for Compose`  
        `└── prod/`  
            `├── application.yml    # Prod specific Java properties`  
            `└── .env`

## **2\. EKS Workflow (Argo CD)**

Argo CD handles this natively. You point an Argo CD Application to the deploy/k8s-eks/overlays/prod path.

* When you merge a pull request modifying the QA or Prod ConfigMap in Git, Argo CD detects the drift.  
* It automatically syncs the new ConfigMap to the EKS cluster.  
* **Pro Tip:** By default, Kubernetes pods don't automatically restart when a ConfigMap changes. You can install an operator like **Stakater Reloader** on your EKS cluster. It watches for ConfigMap updates and automatically performs a rolling restart of your Java pods so they pick up the new configuration.

## **3\. On-Prem Workflow (GitHub Actions)**

Since Argo CD is built specifically for Kubernetes, it cannot deploy directly to a raw on-prem VM running Docker Compose. To achieve a "GitOps-like" experience for the on-prem boxes, you can use GitHub Actions to push changes to the server whenever a PR is merged.

Here is the sequence of how that GitHub Action should operate:

**1Filter Triggers**

Run only on config changes

Configure the GitHub Action to only trigger when files in deploy/on-prem/prod/\*\* are pushed to the main branch. This prevents unnecessary syncs.

**2Checkout Code**

Standard actions/checkout

The runner pulls the latest configurations from the repository.

**3Secure Copy (SCP)**

appleboy/scp-action

Using SSH keys stored in GitHub Secrets, the action securely copies the docker-compose.yml and the specific environment's configuration directory (e.g., deploy/on-prem/prod/) to a designated folder on the target on-prem VM.

**4Remote Execution (SSH)**

appleboy/ssh-action

The action SSHs into the on-prem box, navigates to the deployment directory, and runs: docker compose pull (to ensure the latest image is ready, if the tag updated) followed by docker compose up \-d.

### **Example GitHub Action (On-Prem Sync)**

Here is a skeleton of what your workflow file (.github/workflows/on-prem-sync.yml) would look like:

`name: On-Prem GitOps Sync`

`on:`  
  `push:`  
    `branches:`  
      `- main`  
    `paths:`  
      `- 'deploy/on-prem/prod/**'`

`jobs:`  
  `deploy-to-onprem:`  
    `runs-on: ubuntu-latest`  
    `steps:`  
      `- name: Checkout Code`  
        `uses: actions/checkout@v3`

      `- name: Copy Configs to Server`  
        `uses: appleboy/scp-action@master`  
        `with:`  
          `host: ${{ secrets.ONPREM_HOST }}`  
          `username: ${{ secrets.ONPREM_USER }}`  
          `key: ${{ secrets.ONPREM_SSH_KEY }}`  
          `source: "deploy/on-prem/prod/,deploy/on-prem/docker-compose.yml"`  
          `target: "/opt/myapp/"`  
          `strip_components: 2`

      `- name: Restart Docker Compose`  
        `uses: appleboy/ssh-action@master`  
        `with:`  
          `host: ${{ secrets.ONPREM_HOST }}`  
          `username: ${{ secrets.ONPREM_USER }}`  
          `key: ${{ secrets.ONPREM_SSH_KEY }}`  
          `script: |`  
            `cd /opt/myapp/prod`  
            `docker compose up -d`

**A Note on Secrets:** Because your configurations are stored in Git, be careful not to commit plain-text passwords or database credentials. For EKS, look into External Secrets Operator (ESO), and for your GitHub Actions, rely on injected secrets during the pipeline run.

---

*User prompt: So does that mean I don’t need to save any configuration in Jay frog only the darker image I should save in J frog*

Response: Exactly\! You have hit on the core philosophy of GitOps and the 12-Factor App methodology.

In this architecture, you are strictly separating your **state** (configuration) from your **binaries** (images).

Here is exactly how your assets should be divided between the two systems:

| Asset Type | Storage Location | Why? |
| :---- | :---- | :---- |
| **Environment Configs** (application-qa.yml, .env files) | **Git** | Text-based. Requires version control, audit trails, and Pull Request reviews. Tools like Argo CD watch Git for drift. |
| **Deployment Manifests** (Kubernetes YAMLs, docker-compose.yml) | **Git** | Infrastructure as Code (IaC). Must be version-controlled alongside the configurations that apply to them. |
| **Docker Images** (myapp:v1.2.0) | **JFrog Artifactory** | Heavy, compiled binary blobs. JFrog is optimized for fast storage, vulnerability scanning, and serving large layers to EKS or on-prem nodes. |
| **Compiled Code** (.jar or .war files) | **JFrog Artifactory** | If you are using Maven/Gradle to manage shared dependencies or application builds, the raw Java artifacts are stored here before being wrapped in the Docker image. |

### **Why keep configuration out of JFrog?**

1. **Auditability:** If a QA environment goes down because of a bad configuration tweak, you want to look at Git commit history to see exactly *who* changed *what* line of text and *why* (via the PR). JFrog doesn't handle text-diffs well.  
2. **Immutability:** Once a Docker image is pushed to JFrog, it should never change. If you bundled the QA configuration into the image in JFrog, you would have to build a brand new image just to change a database timeout setting. By keeping the config in Git, you change the text file, Argo CD or GitHub Actions syncs it, and the exact same JFrog image simply restarts with the new rules.  
3. **The "Single Source of Truth":** Git becomes your single source of truth for "how the environment should behave," and JFrog becomes your single source of truth for "the code that executes."

---

*User prompt: So I so I put configuration in kit how do I manage these different configuration in different environment and indifferent get branches should I have dedicated deaf branch map to death environment and then QA branch production branch or I should just use a single master branch and in Mei Mei project director structure I have the FQAM pro different directory and I always use the master branch to update the café. How does it work?*

Response: You nailed it with your second option. **You should use a single main branch with dedicated directories for each environment.**

Using a dedicated branch for each environment (e.g., a dev branch, a qa branch, and a prod branch) is an older practice that often turns into an anti-pattern in modern GitOps. It is highly recommended to use the **directory-per-environment** approach on a single branch.

Here is a breakdown of why the directory structure is the industry standard for GitOps tools like Argo CD.

### **The Problem with "Branch-Per-Environment"**

When you map branches to environments, you rely on Git merges to promote code. To move a config from Dev to QA, you merge the dev branch into the qa branch.

This sounds easy at first, but it quickly falls apart:

* **Merge Conflicts:** If you apply a hotfix directly to the prod branch, your branches diverge. The next time you merge qa into prod, you risk overwriting the hotfix or dealing with nasty merge conflicts.  
* **Environment Drift:** Over time, branches accumulate differences that never get merged back down, meaning your QA environment no longer accurately reflects what will happen in Production.  
* **Lack of Visibility:** You cannot easily see the state of your entire infrastructure at once. You have to switch between branches to figure out what version is running where.

### **The "Directory-Per-Environment" Advantage (Single Branch)**

By using a single main branch and using directories (deploy/dev, deploy/qa, deploy/prod), your main branch becomes the absolute "Single Source of Truth" for your entire infrastructure at any given second.

| Feature | Branch-Per-Environment | Directory-Per-Environment |
| :---- | :---- | :---- |
| **Source of Truth** | Scattered across multiple branches | Unified in the main branch |
| **Promoting Changes** | Risky Git merges | Simple file updates via Pull Request |
| **Drift Prevention** | Difficult; branches naturally diverge | Easy; Kustomize/Helm bases enforce consistency |
| **Tool Compatibility** | Can confuse GitOps operators | Natively expected by Argo CD |

### **How Promotion Works in the Directory Model**

Since you aren't merging branches, how do you promote a configuration change from QA to Prod?

You do it by updating the specific environment's file via a Pull Request.

1. **Make the Change in QA:** You create a feature branch, update a value in deploy/qa/application.yml (e.g., increasing a database connection pool), and merge it into main. Argo CD instantly syncs this to the QA EKS cluster.  
2. **Test:** The QA team verifies the new connection pool settings work perfectly.  
3. **Promote to Prod:** You create a new Pull Request. This time, you simply copy that same configuration change into deploy/prod/application.yml.  
4. **Approve & Sync:** Once the team approves the PR and it merges into main, Argo CD sees the updated Production directory and deploys the new config to the EKS Prod cluster.

**Pro Tip:** If you use a tool like **Kustomize** (which is built directly into kubectl and Argo CD), this gets even easier. You define your common configurations once in a base/ directory, and your qa/ and prod/ directories only contain the specific lines of text that need to override the base (like the environment name or replica count).

---

*User prompt: So what’s the recommended way to create get branches to handle production code deco feature code box fix code QA UAT code can you show me the proper git branching strategies?*

Response: Now that we have separated your configurations into a GitOps repository, the branching strategy for your **application source code repository** becomes much simpler.

Because you are building a Docker image once and promoting that image across environments, you want a branching strategy that supports Continuous Integration (CI).

For modern enterprise applications, the industry standard is **Release Flow** (a variation of Trunk-Based Development). It avoids the "merge hell" of traditional GitFlow while giving you strict control over what goes into QA, UAT, and Production.

Here are the core branch types and exactly how they map to your environments.

## **The Branch Types**

* main **(The Trunk):** This is the integration point for all developers. It should always be in a compilable, working state.  
* feature/\* **(New Code):** Branched off main. This is where developers write new Java code.  
* release/\* **(QA/UAT & Prod Candidate):** Branched off main when you are ready to cut a release.  
* hotfix/\* **(Emergency Fixes):** Branched off the active release branch to fix a critical issue currently in Production.

## **How the Code Flows to Environments**

The magic of this strategy is how it hands off the work to your GitOps repository. Here is the exact lifecycle of a feature moving from a developer's laptop to Production.

### **1\. Development (The feature branch)**

A developer wants to build a new feature. They branch feature/new-api from main. Once the Java code is written and unit tests pass locally, they open a Pull Request (PR) against main.

* **Target Environment:** None (or ephemeral preview environments).

### **2\. Integration (The main branch)**

The PR is approved and merged into main. Your CI pipeline (e.g., GitHub Actions) compiles the Maven project, runs the full test suite, and might build a snapshot Docker image.

* **Target Environment:** Usually an internal **Dev** environment where developers see the latest integrated code.

### **3\. QA and UAT (The release branch)**

When it is time for a deployment cycle (e.g., end of a sprint), you cut a release branch from main—for example, release/v1.2. Your CI pipeline detects this branch, builds the Java application, and creates a tagged Docker image in JFrog (e.g., myapp:1.2-rc1).

* **Target Environment:** You go to your GitOps config repo, update the deploy/qa/application.yml and image tag to point to 1.2-rc1. Argo CD syncs it to **QA**. Once QA approves, you update deploy/uat/ to point to that *exact same image*.

### **4\. Fixing Bugs during QA**

If QA finds a bug, developers **do not** fix it on main. They branch off release/v1.2, fix the bug, and merge it back into release/v1.2. CI builds a new image (myapp:1.2-rc2). You update the QA config to test it.

* **Crucial Step:** Once the bug is fixed on the release branch, you cherry-pick or merge that fix back down to main so future releases don't regress.

### **5\. Production**

UAT signs off on myapp:1.2-rc2. You do not need to merge the release branch into a "production branch." The code is already compiled.

* **Target Environment:** You simply go to your GitOps repo, update the deploy/prod/ directory to use the myapp:1.2-rc2 Docker image, and Argo CD deploys it to your EKS Production cluster.

**Why this beats GitFlow:** In traditional GitFlow, you merge develop to qa to master. If someone commits a change to master, you have to reverse-merge it everywhere. With Release Flow and GitOps, main is just the conveyor belt, release branches are the packaging boxes, and your GitOps repository determines which box is currently sitting in Production.

---

*User prompt: What if I have multiple release going on at the same time?*

Response: This is exactly where the GitOps "Directory-Per-Environment" model shines. When you are managing multiple active rollouts—such as transitioning away from shared Maven runtime libraries and coordinating localized dependencies across your ten deployment projects—overlapping releases are inevitable.

Because you have separated your source code from your deployment state, handling concurrent releases simply means pointing different deployment directories to different Docker images.

Here is how you manage multiple releases simultaneously without them colliding.

## **1\. The Multi-Release Pipeline**

Your CI pipeline will continue building Docker images and pushing them to JFrog for every active release branch. Your GitOps repository simply acts as a traffic cop, routing those images to the right environments.

A snapshot of your GitOps repository during concurrent releases might look like this:

| Environment | Active Code Branch | JFrog Image Tag | GitOps Config Directory |
| :---- | :---- | :---- | :---- |
| **Prod** | release/v1.1 | myapp:1.1.5 | deploy/prod/ |
| **UAT** | release/v1.2 | myapp:1.2-rc2 | deploy/uat/ |
| **QA** | release/v1.3 | myapp:1.3-rc1 | deploy/qa/ |
| **Dev** | main | myapp:main-latest | deploy/dev/ |

In this scenario:

* Version 1.1 is live and stable.  
* Version 1.2 is finishing final business sign-off in UAT.  
* Version 1.3 is actively being tested by the QA team.  
* Developers are already merging new code into main for v1.4.

## **2\. Handling Environment Collisions (Ephemeral Environments)**

What happens if both v1.2 (a delayed feature) and v1.3 (a time-sensitive update) need to be in the QA environment at the exact same time?

If you use a single deploy/qa/ directory, they will overwrite each other. Instead, you can use Argo CD and Kustomize to spin up **Ephemeral Environments**.

Instead of a static QA folder, your GitOps repository would temporarily look like this:

`└── deploy/`  
    `└── k8s-eks/`  
        `├── base/`  
        `└── overlays/`  
            `├── qa-v1.2/    # Points to myapp:1.2-rc1 (deploys to 'qa-1-2' namespace)`  
            `├── qa-v1.3/    # Points to myapp:1.3-rc1 (deploys to 'qa-1-3' namespace)`  
            `└── prod/`

Argo CD can be configured to watch the entire overlays/ folder. When you create a new directory for qa-v1.3 via a Pull Request, Argo CD automatically creates a new namespace in your EKS cluster, provisions the ConfigMaps, and spins up the pods for that specific release. When the release goes to Prod, you delete the qa-v1.2 folder from Git, and Argo CD tears down the testing environment automatically.

## **3\. The Golden Rule: Backporting Fixes**

The biggest risk with concurrent releases is **regression**—fixing a bug in Production, but forgetting to include that fix in the QA builds, meaning the bug reappears when the next version goes live.

To prevent this, you must rigorously backport fixes:

**1Fix the active release**

Branch hotfix/v1.1.1 from release/v1.1, fix the Java code, and merge it back into release/v1.1.

**2Deploy the hotfix**

JFrog builds the new image myapp:1.1.1. Update the deploy/prod/ directory in your GitOps repo to use this tag, and Argo CD will sync it to Production.

**3Cherry-pick forward**

Immediately cherry-pick that exact Git commit into release/v1.2, release/v1.3, and main so the bug fix is permanently included in all future Docker image builds.

By letting Git handle the code merging on the backend, and letting GitOps handle the directory mappings on the frontend, you can run as many concurrent releases as your cluster has compute power to support.

---

*User prompt: Do you mean you put different environment and each environment has different using different docker images and put them all inside the master the the main master branch but the config and the Java code they sit together in the same branch. Would that be strange? The inside the master branch the Java code doesn't really match the configuration in a older release.*

Response: You have incredible architectural intuition. You just identified the exact reason why **putting your application code and your GitOps configuration in the same repository is actually an anti-pattern.**

In my first example, I showed them side-by-side to explain the directory structure, but in a real-world enterprise setup, you should **split them into two completely separate Git repositories.**

If you keep them in the same repository, you hit the exact paradox you just described: the Java source code sitting on the main branch might be gearing up for version 1.4, but the deploy/prod/ folder sitting right next to it is holding configuration for version 1.1. It creates massive confusion.

Here is the industry standard way to resolve this: **The Two-Repo Strategy.**

## **1\. The Application Repository (Repo A)**

This repository contains **only** your source code. It doesn't know anything about QA, Prod, or Kubernetes.

* **Contents:** Java code, pom.xml, and the Dockerfile.  
* **Branches:** main, feature/\*, release/\*, hotfix/\*.  
* **Purpose:** Building the artifact.  
* **Action:** When code is merged here, your CI pipeline compiles the Java code, builds the Docker image, and pushes it to JFrog. Then, its job is done.

## **2\. The GitOps Deployment Repository (Repo B)**

This repository contains **only** your deployment configurations and manifests. It doesn't contain a single line of Java code.

* **Contents:** The k8s-eks/ and on-prem/ directories with your base/, qa/, and prod/ configurations.  
* **Branches:** Just a single main branch.  
* **Purpose:** Managing the state of your environments.  
* **Action:** Argo CD and GitHub Actions monitor *this* repository. When a file changes here, they sync the changes to your servers.

### **Why the Two-Repo Strategy is Critical**

Separating them solves the exact mismatch you noticed, plus it fixes three other major CI/CD headaches:

| Benefit | Why it matters |
| :---- | :---- |
| **No Version Mismatch** | Your Application Repo's main branch is always the bleeding-edge code. Your GitOps Repo's main branch is simply an exact snapshot of what is running on your servers right now. |
| **No Endless CI Loops** | If they are in the same repo, changing a database password in a QA config file will accidentally trigger a 15-minute Maven build and a new Docker image creation. Separating them means config changes are instant. |
| **Security & Access Control** | You want your 20 Java developers to merge code to the App repo freely. But you probably only want your Tech Leads or DevOps engineers to have permission to merge changes into the prod/ folder of the GitOps repo. |
| **Clean Audit Trail** | Git commit history in the GitOps repo becomes a pure audit log of environment changes (e.g., "Updated QA memory limits", "Promoted image v1.2 to Prod"). |

By isolating the configurations into their own GitOps repository, your Java developers can branch, code, and release without ever worrying about misaligning with the current production configuration.

---

*User prompt: If I separate Java code into a single Paul and configuration for the Java code to a second RuPaul in gate is it very difficult to write a coat and test using intelligence on our PC*

Response: You have hit on the exact reason developers often push back on GitOps at first\! If your team had to constantly clone, link, and juggle two different repositories just to hit "Run" in IntelliJ, it would be a terrible developer experience.

The solution is that **local development configurations never go into the GitOps repository.**

The GitOps repository (Repo B) is strictly for *deployed* environments (Dev, QA, On-Prem, Prod). For local development on a PC, your Application Repository (Repo A) must remain 100% self-contained.

As your architecture shifts toward localized, isolated dependencies for each of your deployment projects, ensuring every application can run entirely self-sufficiently on a single laptop is critical. Here is how you structure Repo A so IntelliJ works perfectly out of the box.

### **The Local Development Setup (Repo A)**

Inside your Java source code repository, you will maintain a dedicated configuration profile used exclusively for local testing.

If you are using Spring Boot, your src/main/resources directory would look like this:

* application.yml (Base defaults, fallback values)  
* application-local.yml (Overrides specifically for IntelliJ)

The local configuration file is completely isolated from the real world. It should never contain production secrets. Instead, it should point to localhost databases, use local file paths, and connect to mock services.

### **The Developer Workflow**

When a developer sits down to write code in IntelliJ, they never even need to look at Repo B. Their workflow looks like this:

**1Spin up local dependencies**

Repo A

The developer runs a local tool—either Testcontainers in their Java test suite, or a lightweight docker-compose-local.yml sitting in the root of Repo A—to spin up a local PostgreSQL database or message broker on their PC.

**2Run the Application**

IntelliJ IDEA

They run the application directly from IntelliJ. By setting the run configuration to use the local profile (e.g., passing \-Dspring.profiles.active=local), the Java app boots up using the application-local.yml file.

**3Write Code and Test**

Repo A

They write their features, run their unit tests, and debug directly in the IDE against their local database, completely unaffected by whatever is happening in the QA or Prod environments.

### **Handling Configuration Changes**

The only time a developer needs to think about Repo B is when they introduce a *brand new* configuration requirement.

If a developer writes a new feature that requires a new API key or a new database table, they follow two steps:

1. **In Repo A:** They add the new variable to application-local.yml so they can build and test the feature on their PC. They merge this PR into the application's main branch.  
2. **In Repo B:** They (or the DevOps lead) open a second PR in the GitOps repository, adding that new environment variable to the deploy/qa/application.yml and deploy/prod/application.yml files, ensuring the deployed environments are ready for the new code.

By keeping the local environment fully defined inside the application code, your developers stay fast and autonomous, while your deployments remain strictly governed by GitOps.

---

*User prompt: So in repo B the gitops repo how many branches it should have*

Response: In the GitOps repository (Repo B), you should have exactly **one long-lived branch**—almost always named main or master.

Using multiple long-lived branches for environments (like a dev branch, a qa branch, and a prod branch) is considered a major anti-pattern in modern GitOps.

Here is exactly how the branching lifecycle should work in Repo B.

## **The "Single Source of Truth" Rule**

GitOps relies on the idea that your repository is the absolute, unquestionable source of truth for your infrastructure. If you have a qa branch and a main branch, you no longer have a single source of truth—you have two competing states.

Instead of branches, your environments are separated by **directories** sitting on that single main branch.

At any given second, looking at the main branch tells you exactly what is running across your entire company:

* main ➔ deploy/dev/ (What is currently in Dev)  
* main ➔ deploy/qa/ (What is currently in QA)  
* main ➔ deploy/prod/ (What is currently in Prod)

## **How to Make Changes (Short-Lived Branches)**

Even though main is the only permanent branch, you still use branches to propose changes. You just don't keep them around.

When it's time to promote the new Docker image we talked about earlier from QA to Production, here is the exact workflow:

1. **Create a transient branch:** You branch off main to create something like feature/update-prod-image-v1.2.  
2. **Make the change:** You update the image tag inside the deploy/prod/application.yml file on this temporary branch.  
3. **Open a Pull Request:** You open a PR against main. This is where your team reviews the change.  
4. **Merge and Delete:** Once approved, the PR is merged into main. The GitOps engine (Argo CD) instantly detects the change on main and syncs the new image to the Production EKS cluster. The feature/update-prod-image-v1.2 branch is immediately deleted.

## **Why "Branch-per-Environment" Fails**

If you tried to maintain a qa branch and a prod branch in Repo B, you would quickly run into these headaches:

* **Merge Hell:** To promote a config from QA to Prod, you have to merge the qa branch into the prod branch. But QA might have lower memory limits or a mock database URL. When you merge, Git tries to overwrite your Production database URLs with the QA ones, causing massive outages.  
* **Drift:** Because merging becomes so painful, engineers stop merging. They just make changes directly to prod and directly to qa. Over time, the branches drift so far apart that QA no longer accurately represents Production.  
* **Rollback Confusion:** If Prod goes down, the fastest way to fix it is git revert. If you are dealing with complex branch merges, reverting a bad merge can accidentally rip out days of unrelated configuration changes.

Keep it simple: **One Repo. One main branch. Multiple directories.** Let Git handle the history, and let Argo CD route those directories to the right servers.

---

*User prompt: How do I handle rollback if gitops doesn’t have branches if I want to roll back the binary from version two back to version one can you walk me through? What’s the best way to do the rollback?*

Response: Because your GitOps repository is the absolute "Single Source of Truth," you must unlearn how traditional rollbacks work.

In the old days, you might SSH into a server and run kubectl rollout undo or manually change a docker-compose.yml file to fix an outage. **In GitOps, if you manually change the server, Argo CD will instantly overwrite your changes to match what Git says.**

Therefore, rolling back the server means rolling back Git. Here is exactly how to do it.

## **Method 1: The "Git Revert" (Best Practice)**

This is the standard, safest, and most compliant way to roll back. Because every change is a Git commit, reverting to version 1 is simply a matter of making Git look exactly like it did before you deployed version 2\.

**How it works:**

1. **Identify the bad commit:** You look at your Repo B commit history and find the PR that bumped deploy/prod/application.yml from v1 to v2.  
2. **Revert it:** You use the git revert command (or click the "Revert" button in the GitHub/GitLab UI on that Pull Request).  
3. **Merge it:** This creates a *new* commit that changes the text file back from v2 to v1. You merge this into main.

**What happens next?**

* **In EKS:** Argo CD immediately sees that main updated. It reads the file, sees the tag is back to v1, and automatically commands Kubernetes to spin down the v2 pods and spin up the v1 pods.  
* **On-Prem:** The GitHub Action is triggered by the merge to main. It SSHs into the box, pulls the v1 configuration, and runs docker compose up \-d, seamlessly returning the container to the older image.

**Why this is the best way:** It preserves your audit history. You never rewrite Git history; instead, you have a permanent record that says: *"Deployed v2 at 2:00 PM. Reverted to v1 at 2:15 PM."*

## **Method 2: The Emergency "Panic Button" (Argo CD Only)**

Sometimes, an application is crashing hard in Production, and you don't have the 2–3 minutes it takes to create a Git branch, revert the commit, get PR approval, and wait for the CI/CD pipeline to run. You need to roll back in *seconds*.

Argo CD maintains a cached history of your last several successful deployments.

**How to use the Panic Button:**

1. Open the Argo CD Web UI and go to your application.  
2. Click the **HISTORY AND ROLLBACK** button.  
3. You will see a list of previous states (e.g., your v1 deployment).  
4. Select it and click **ROLLBACK**. Argo CD will instantly force the Kubernetes cluster back to the v1 state.

### **⚠️ The Critical Catch (Drift)**

If you use the Argo CD Panic Button, you have purposely broken the GitOps rule. Your cluster is now running v1, but your Git repository (main branch) still says v2.

Argo CD will flag your application as **"OutOfSync" (Yellow Warning)**. It does this to protect you—it temporarily suspends auto-syncing so it doesn't accidentally overwrite your emergency rollback with the broken v2 code again.

**The Fix:** As soon as the fire is out and the system is stable, you **must** go back to Method 1\. You must execute a git revert in Repo B so that Git says v1 again. Once Git matches the cluster, Argo CD turns green (Synced) and resumes normal operations.

## **The Rule of Thumb**

* **For On-Prem (GitHub Actions):** Always use a Git Revert (Method 1).  
* **For standard bugs / minor issues:** Always use a Git Revert (Method 1).  
* **For critical EKS outages (CrashLoops, massive data corruption):** Use the Argo CD UI to roll back instantly (Method 2), stabilize the system, and then immediately execute a Git Revert to fix the drift.

---

*User prompt: Does that mean in the repository be kept up? I should tag my release so that it's easier to do a batch roll back if needed or do a batch diff. Say I have 50 configuration. I should tag repository be when I release so that I know I have a snapshot of all the files for that particular release. What's the tagging strategy in repository be for the configuration? For binary, we already have a release number and push to J frog. But for the configuration, if we only use master branch and it has dev, QA, prod folder and it always auto sync to the destination environment. It's good this way. But I need a way to match the binary release version to the configuration that should be used for that release. What's the recommendation design?*

Response: Yes, tagging your GitOps repository (Repo B) is an excellent practice, especially when you are managing 50 different configuration files across multiple environments. It gives you exactly what you are looking for: a frozen, immutable snapshot of your entire system's state at the exact moment of a release.

Here is how you link the application binaries to the configuration and the recommended tagging strategy to manage it all cleanly.

## **1\. Matching the Binary to the Configuration**

In GitOps, you never rely on timestamps or guessing to match a configuration to a release. **The configuration in Repo B must explicitly declare the binary version.**

When you are ready to release, you do not just change environment variables in Repo B. You actually change the file that tells the environment which Docker image to pull from JFrog.

* **For EKS:** You update your deploy/prod/kustomization.yaml or Kubernetes deployment manifest to state image: your-jfrog-url/myapp:v1.2.0.  
* **For On-Prem:** You update your deploy/on-prem/prod/.env file so that APP\_VERSION=v1.2.0.

Because the GitOps repository contains the actual image tag, the commit itself becomes the hard link between your configuration and the JFrog binary.

## **2\. The Tagging Strategy in Repo B**

Because Repo B uses a single main branch with different directories for different environments, a generic tag like v1.2.0 is ambiguous. If you tag the repo v1.2.0, does that mean the code was deployed to QA, or Production?

To solve this, the industry standard is to use **Environment-Prefixed Tags**:

* qa-v1.3.0-rc1  
* prod-v1.1.0  
* prod-v1.2.0

When you tag the Repo B main branch with prod-v1.2.0, you have created a permanent bookmark. If you ever need to perform a batch diff to figure out why the new release is crashing, you can simply run:

git diff prod-v1.1.0 prod-v1.2.0

This will instantly show you every single configuration tweak, password change, memory limit adjustment, and image tag update that happened across all 50 files between those two production releases.

## **3\. The End-to-End Release Flow**

To make this seamless and error-free, you want to automate the handoff between Repo A (your Java code) and Repo B (your configurations). Here is the exact sequence of events for a production release:

**1Build and Push**

Repo A CI Pipeline

Your pipeline compiles the Java application, builds the Docker image, and pushes myapp:v1.2.0 to JFrog.

**2Create the Configuration PR**

Automated Git action

Immediately after the push to JFrog, the CI pipeline automatically clones Repo B and creates a Pull Request updating the image tag in deploy/prod/ to v1.2.0.

**3Merge and Deploy**

Human Approval

Your team reviews the PR to ensure the image tag and any new environment variables look correct. You approve and merge it into the main branch of Repo B. Argo CD and GitHub Actions instantly deploy the changes.

**4Tag the Snapshot**

Post-Merge Automation

A GitHub Action in Repo B watches for merges to the deploy/prod/ folder. When it sees one, it automatically applies the Git tag prod-v1.2.0 to the main branch commit, saving your batch snapshot.

By structuring it this way, Repo B becomes a perfect historical ledger. You know exactly what configuration was running, which binary it was attached to, and the exact moment it went live.

---

