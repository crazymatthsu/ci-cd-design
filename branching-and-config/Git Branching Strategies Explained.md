# **Git Branching Strategies Explained**

[https://gemini.google.com/app/f38ae43ceff03c8b](https://gemini.google.com/app/f38ae43ceff03c8b)

*User prompt: if the team lacks of unit tests, lacks of integration tests, and mainly rely on human QA/UAT testing, which branching strategies should be used ?*

### **Response: The Direct Answer**

Under these specific conditions, you must absolutely avoid Trunk-Based Development. When coordinating complex deployments across multiple teams or managing shared libraries without the safety net of automated testing, you should use **Git-Flow** or an **Environment-Based Branching** strategy.

### **Why Trunk-Based Development Will Fail Here**

Trunk-Based Development relies entirely on a robust, fast automated testing pipeline (CI) to act as an immediate gatekeeper.

> * **Broken Mainline:** Without unit and integration tests catching regressions instantly, buggy code will be merged into your main branch daily.  
> * **Blocked Testing:** Manual QA takes time. If main is constantly receiving new, untested commits, the QA environment will be highly unstable, making it impossible for humans to verify features reliably.  
> * **Deployment Paralysis:** Because the trunk is never guaranteed to be in a deployable state, you lose the ability to release software with confidence.

### **Recommended Strategies**

#### **1\. Git-Flow (Best for Scheduled Releases)**

Git-Flow is built specifically to accommodate long manual testing cycles and stabilization periods.

> * **Dedicated Release Branches:** When it is time for QA or UAT to begin, you cut a release/\* branch from develop.  
> * **Isolated Stabilization:** QA can spend days or weeks testing the release branch. Any bugs found are fixed directly on that branch, keeping the code frozen from new feature additions.  
> * **Unblocked Developers:** While QA tests the release branch, developers can continue merging entirely new features into develop without destabilizing the version QA is actively evaluating.

#### **2\. Environment-Based Branching (Best for Pipeline Mapping)**

If your infrastructure is strictly tied to physical environments, mapping your branches to these environments provides very clear boundaries for manual testers.

> * **The DEV/Integration Branch:** Developers merge features here. It is expected to be relatively unstable and is used for initial integration.  
> * **The UAT/QA Branch:** Code is promoted here only when it is ready for human testing. This branch is locked down and stable, giving your QA team a reliable target.  
> * **The PROD (Main) Branch:** Code only reaches this branch after formal human sign-off in the UAT environment.

### **The Path Forward**

While these strategies will protect your production environment today by isolating untested code, relying entirely on manual human verification scales poorly and often leads to massive deployment bottlenecks as the application architecture grows.

Are you looking to eventually transition this team toward more automated GitOps pipelines, or is the immediate goal simply to stabilize their current manual QA workflow?

---

*User prompt: for git flow strategy, if I have 5 feature branches from main , can I merge 5 features into dev-1 branch, if I want to pull out a feature, I can delete dev-1 branch , and create a dev-2 branch by merging 4 other features into dev-2 branch, what's the feature back out strategies recommended*

Response: Your proposed method—often referred to as **Throwaway Integration Branches** or **Ephemeral Environments**—is actually a highly effective and widely used pattern.

When you are coordinating deployments, schemas, and shared libraries across multiple trading teams, keeping the commit history clean and predictable is critical. Your approach completely avoids the messy commit history that traditional Git reverts leave behind.

Here is a breakdown of your proposed strategy alongside the other recommended feature backout strategies.

### **1\. Throwaway Integration Branches (Your Proposed Strategy)**

Instead of treating your dev branch as a permanent, long-lived fixture, you treat it as a temporary container for a specific release candidate.

> * **The Workflow:**  
  1. Cut dev-1 from main.  
  2. Merge Features A, B, C, D, and E into dev-1.  
  3. QA finds a critical flaw in Feature C.  
  4. Delete dev-1.  
  5. Cut dev-2 from main.  
  6. Merge Features A, B, D, and E.  
> * **Pros:** It guarantees a pristine Git history. There are no leftover artifacts, partial rollbacks, or confusing revert commits. You know with 100% certainty that Feature C's code is not in the release.  
> * **Cons:** If Features A, B, D, and E had overlapping code changes, you will have to resolve the exact same merge conflicts again when creating dev-2.

### **2\. The git revert Strategy**

This is the traditional Git method for backing out a feature that has already been merged into a shared integration branch.

> * **The Workflow:** You identify the merge commit where the unwanted feature was brought into the dev branch and run a revert command (e.g., git revert \-m 1 \<merge-commit-hash\>). This creates a *new* commit that applies the exact inverse of the feature's changes.  
> * **Pros:** You do not have to delete and recreate branches or re-resolve previous merge conflicts from the other four features.  
> * **Cons (The "Revert-the-Revert" Trap):** Git now records that the feature was merged, and then explicitly rejected. If the developer fixes Feature C a week later and tries to merge it back into dev, Git will ignore the original commits because it sees they were already merged and reverted. To get the feature back in, the developer has to mathematically revert the revert commit, which is highly unintuitive and error-prone.

### **3\. Feature Flags (The Architectural Strategy)**

Rather than using Git to manage what features are released, you merge all five features into the dev branch, but wrap the new code paths in configuration toggles.

> * **The Workflow:** Feature C is merged into dev-1 and deployed to the QA environment. When it fails QA, no Git operations are performed. Instead, an environment variable or configuration flag for Feature C is flipped to OFF.  
> * **Pros:** Completely decouples deployment from release. Backing out a feature takes seconds and requires no code changes, branch deletions, or CI/CD pipeline reruns.  
> * **Cons:** Requires upfront architectural planning to implement the toggles. It also introduces technical debt; once a feature is permanently launched, developers must remember to go back and strip the toggle logic out of the codebase.

Are the features your teams are merging heavily dependent on overlapping schemas and files, or are they isolated enough that recreating the integration branch will generally be free of major merge conflicts?

---

*User prompt: in order to use feature flags, the code path needs to be in a new code path , without affecting any existing code path , correct ?*

Response: **Yes, logically—but with an important architectural nuance.**

To safely use a feature flag, the **runtime execution** of your existing system must remain 100% unaffected when the flag is turned OFF. If the flag is disabled, the application must execute the exact same fallback behavior as it did before the new deployment, with zero side effects.

However, at the **code level**, you usually have to touch existing code paths to introduce the "decision point" (the routing logic that checks the flag). The goal is to make that decision point as clean and isolated as possible so the new logic doesn't bleed into or destabilize the legacy logic.

Here is how that architectural isolation is typically achieved in practice, ranging from simple changes to complex backend systems.

### **1\. Simple Conditional Routing (if / else)**

For minor logic changes or UI tweaks, you modify the existing code path just enough to insert a conditional check. The new code is encapsulated inside its own block, while the else block preserves the existing code path entirely untouched.

// Existing code path is modified only to add the routing check  
if (FeatureFlag.isEnabled("NEW\_ALGO\_V2")) {  
    // NEW code path: completely isolated  
    return executeNewAlgorithm(data);  
} else {  
    // LEGACY code path: executes exactly as it did before  
    return executeLegacyAlgorithm(data);  
}

### **2\. Branch by Abstraction (The Strategy Pattern)**

Scattering if / else statements across complex, high-throughput business logic or core processing pipelines is dangerous and creates technical debt. For larger enterprise systems, the recommended pattern is **Branch by Abstraction**.

Instead of modifying the internal logic of an existing code path, you pull the behavior behind an interface and use the feature flag at the factory or dependency injection level to route traffic.

> * **Step 1:** Define an interface (e.g., OrderValidationStrategy).  
> * **Step 2:** Wrap your current existing code untouched into a LegacyOrderValidator class that implements the interface.  
> * **Step 3:** Build your new code in a completely new, isolated file: NewOrderValidator.  
> * **Step 4:** At the application root or service router, use the feature flag to decide which implementation to instantiate and inject:

OrderValidationStrategy validator \= FeatureFlag.isEnabled("VAL\_V2")   
    ? new NewOrderValidator()   
    : new LegacyOrderValidator();

// The underlying business logic calls the interface without knowing or   
// caring which code path is executing.  
validator.validate(order);

**Why this matters:** If a critical bug is found in NewOrderValidator, toggling the flag back to OFF immediately redirects execution back to LegacyOrderValidator without ever risking the structural integrity of the legacy code.

### **3\. The Exception: Where "New Code Paths" Are Not Enough**

The principle of isolating new code paths gets significantly harder when the feature interacts with **shared state**, such as database schemas, messaging payloads, or shared external libraries.

If a new feature relies on a database schema change or an updated API contract, a feature flag in the application code is meaningless if the underlying data structure is incompatible with the legacy code path. In these scenarios, you must combine feature flags with the **Expand and Contract (Parallel Change)** pattern:

> 1. **Expand (Additive Only):** Add the new database table, column, or message field in a backwards-compatible way without touching or removing the old ones. The database or schema now supports *both* states.  
> 2. **Migrate & Toggle:** Deploy the code with the feature flag. When the flag is ON, it reads/writes to the new schema structures. When it is OFF, it safely continues reading/writing to the old structures because they were never modified or deleted.  
> 3. **Contract (Cleanup):** Only after the feature flag is permanently turned ON and verified in production do you perform a final deployment to delete the legacy code path, the feature flag toggle, and the old schema columns.

### **Summary Checklist for Feature Flag Safety**

> * **Zero Side Effects when OFF:** The legacy path must not rely on any new initialization, state changes, or data transformations introduced by the new feature.  
> * **No "Fail-Open" Bugs:** If the feature flag evaluation service (or configuration file) fails to load or becomes unreachable, the application must default hard to OFF (the legacy code path).  
> * **Decoupled Schemas:** Never put a breaking schema or payload change behind a code-level feature flag without first making the underlying data store additive and backwards-compatible.

---

*User prompt: Explain to me the importance of having gradle sub modules to segregate business logic and a common module to have common code layer, why this can help with branching and avoid merge conflicts, and why do we need to use Throwaway Integration Branches strategy due to lack of fully covered unit tests and integration tests, we cannot use trunk based branching strategies, and why automatic testing is the most critical factor to drive branching strategies , I need organized points to present to a team of 20 people we will use Throwaway Integration Branches and not 100% trunk based branching strategies right now, but in the future once we have more complete automatic unit and integration testing , we will move closer to trunk based branching strategies*

Response: Here is a structured breakdown designed to be presented to your engineering team. It logically connects your architectural choices (Gradle submodules) with your testing realities, ultimately justifying the interim branching strategy while providing a clear roadmap for the future.

### **Part 1: Architectural Foundation—Why Gradle Submodules Matter**

To scale development across a 20-person group—especially when coordinating deployments and schemas across different domains like trading algorithms, execution routing, or reporting—the codebase must be structurally partitioned.

> * **Isolating Business Logic:** By breaking the application down into independent Gradle submodules, distinct functional areas are physically separated. Team A can build out new order-routing features in :module-routing, while Team B refactors data pipelines in :module-analytics.  
> * **The Common Code Layer:** Shared utilities (e.g., database connections, distributed messaging clients, or standard security protocols) live in a restricted :module-common. Changes here are deliberate and rare, preventing widespread regressions.  
> * **Drastically Reducing Merge Conflicts:** Merge conflicts happen when developers alter the same files simultaneously. When architecture is heavily modularized, developers are naturally funneled into separate directories and submodules. You eliminate the "merge hell" that occurs when multiple teams collide in a monolithic directory structure.  
> * **Targeted Build and Test Times:** Gradle can detect which submodules have changed and only compile and test those specific modules. This keeps CI pipelines fast, which is a mandatory prerequisite for eventually moving to rapid branching strategies.

### **Part 2: The Testing Imperative—Why Automation Dictates Branching**

Branching strategies are not arbitrary preferences; they are directly dictated by the speed and reliability of a team's testing apparatus.

> * **The CI/CD Safety Net:** In modern deployment pipelines, automated tests are the only gatekeepers. If a system lacks comprehensive unit and integration tests, it lacks an automated safety net.  
> * **The Bottleneck of Human Verification:** Manual QA and UAT are high-fidelity but extremely slow. If a pipeline relies on human verification, integration is inherently delayed by hours or days.  
> * **The Danger of a Broken Mainline:** If a team attempts Trunk-Based Development without automated tests, buggy, unverified code will merge into main daily. The application will be in a constant state of brokenness, blocking QA from testing effectively and paralyzing the team's ability to deploy to production safely.

### **Part 3: Our Current Strategy—Throwaway Integration Branches**

Because our primary validation mechanism relies on manual QA/UAT rather than an automated CI test suite, we cannot safely merge unverified features directly into a shared trunk. We will utilize **Throwaway Integration Branches** as our immediate, pragmatic solution.

> * **Protecting the Mainline:** main remains pristine. It only accepts code that has explicitly passed UAT, ensuring that production deployments are always safe and predictable.  
> * **Accommodating Manual QA:** We compile release candidates by merging targeted features into an ephemeral integration branch (e.g., release-candidate-v1). This branch is deployed to a staging environment, giving QA a stable, isolated target that won't shift under their feet while they test.  
> * **Clean and Painless Backouts:** If QA discovers a critical flaw in one of the aggregated features, we do not waste time untangling complex git revert operations. We simply discard the integration branch, cut a new one from main, and merge only the healthy features. The Git history remains clean, linear, and completely free of technical debt.

### **Part 4: The Roadmap—Evolving to Trunk-Based Development**

Throwaway integration branches are a necessary tactical stepping stone, but they are not the permanent destination. As our engineering practices mature, our branching strategy will evolve.

> * **Step 1: Build the Automated Safety Net:** We will begin systematically increasing our unit test coverage within individual Gradle submodules and implementing automated integration tests for critical business flows.  
> * **Step 2: Implement Feature Flags:** We will introduce architectural routing (feature toggles) to decouple code deployment from feature release, allowing incomplete code to live safely in production without executing.  
> * **Step 3: The Transition to Trunk:** Once our automated pipeline can catch regressions in minutes rather than days, the risk of breaking main drops to near zero. At that threshold, we will retire the manual integration branches and transition to Trunk-Based Development, unlocking true continuous delivery.

To ensure the Gradle submodule structure effectively minimizes those merge conflicts right out of the gate, how tightly coupled are the data schemas currently shared between the different business logic domains?

---

*User prompt: Throwaway integration branch is basically the next release branch , but we are ok to recreate it , correct ?*

### **Response: The Short Answer: Yes**

You have it exactly right. A throwaway integration branch serves the exact same purpose as a traditional release branch—it aggregates the specific features intended for the next deployment so that QA can test them in a unified staging environment.

The defining difference is entirely in its **disposability** and how you handle failure.

### **Traditional Release Branch vs. Throwaway Branch**

To make the distinction crystal clear for your team, here is how the two concepts differ when a bug is inevitably found during UAT:

> * **The Traditional Release Branch (Persistent):** If QA finds a bug in Feature C, developers commit the fix *directly* to the release branch. Because new code was written directly on this branch, it becomes a permanent artifact. After deployment, this branch must be carefully reverse-merged back into main and develop to ensure the bug fixes are not lost.  
> * **The Throwaway Integration Branch (Ephemeral):** It is treated as an immutable, temporary build artifact. If QA finds a bug in Feature C, **no one is allowed to commit fixes to the integration branch.** Instead:  
  1. The developer fixes the bug on the original feature-c branch.  
  2. You delete the broken integration branch entirely.  
  3. You cut a fresh integration branch from main and merge the updated features back in.

### **Why This Mindset Shift is Important**

By defining it as a "throwaway" branch rather than a standard release branch, you force a strict discipline on the team: **Integration branches are for compilation and testing only, never for active development.**

This guarantees that all bug fixes happen at the source (the feature branches), ensuring your Git history remains linear and your main branch never suffers from messy, retroactive merges.

---

*User prompt: so after QA completed, should I create release branch from the 'Throwaway integration branches ' ? and merge release branch back to main after production release ?*

### **Response: The Direct Answer: No**

You do not need to create an additional release branch. Doing so would add an unnecessary "middleman" step that clutters your repository.

Because you treated the **Throwaway Integration Branch** with strict discipline (meaning developers did not commit directly to it, but only merged features into it), this branch inherently serves as your perfect, pristine **Release Candidate (RC)**.

### **The Post-QA Workflow**

Once QA gives the final sign-off on the throwaway integration branch, here is the exact, clean sequence of events you should follow:

**1\. Merge Directly to Main (via Pull Request)** The approved throwaway integration branch (e.g., rc-v1.2.0) is merged directly into your main branch. This is typically done via a Pull Request to ensure there is a formal, documented approval gate before the code hits the production track.

**2\. Tag the Release (Critical for Auditability)** Immediately after merging into main, create a Git tag on that merge commit (e.g., v1.2.0). In environments with strict deployment audits and compliance requirements, this immutable tag is your ultimate source of truth. It explicitly proves exactly what version of the codebase was deployed to production at a specific date and time.

**3\. Deploy to Production** Your deployment pipeline should be triggered by the creation of the tag on the main branch, ensuring that only formally version-controlled, QA-approved code reaches your production environment.

**4\. Delete the Throwaway Branch** Once the code is safely merged to main and tagged, delete the integration branch. It has served its temporary purpose.

### **Why This is the Best Practice**

> * **Linear History:** By merging the integration branch straight to main, your Git history tells a very clear, easy-to-read story: *Feature branches were bundled into an RC \-\> The RC was merged to Main \-\> Main was tagged and deployed.*  
> * **Reduced Branch Sprawl:** If you create a release branch *from* the integration branch, you now have two separate branches representing the exact same state of the code. This causes confusion about which branch is the "true" source of the deployment.  
> * **Zero Retroactive Merging:** You don't have to worry about back-porting fixes from a release branch into main, because the RC branch goes directly to main.

What CI/CD orchestration tool (like Jenkins, GitHub Actions, or GitLab CI) is your team currently using to handle the deployment trigger once main is updated?

---

*User prompt: Is it true : concept of throwaway integration branches is extremely common in modern CI/CD. Today, this pattern is often natively automated by source control platforms under names like "Merge Queues" (GitHub)*

Response: **Yes, this is absolutely true.**

The concept of the throwaway integration branch has proven so effective at keeping the main branch stable that modern source control platforms have built native, automated features around it.

When you use a feature like **Merge Queues** in GitHub (or **Merge Trains** in GitLab), you are essentially using a highly automated, invisible system of throwaway integration branches.

Here is a breakdown of how these native features take the manual labor out of the throwaway integration strategy.

### **How GitHub Merge Queues Automate the Process**

If you have a busy repository where multiple developers are trying to merge features into main at the same time, merging them individually risks "semantic merge conflicts"—where the code merges cleanly via Git, but the combination of the two features breaks the application logic.

A Merge Queue prevents this by automating the throwaway integration branch process:

> 1. **The Queue:** Instead of merging a Pull Request directly into main, a developer clicks "Merge when ready." The PR enters a queue.  
> 2. **The Automated Throwaway Branch:** GitHub silently behind the scenes creates a temporary, hidden integration branch (usually prefixed with something like gh-readonly-queue/main/).  
> 3. **The Bundle:** GitHub bundles the current state of main, plus any PRs ahead of it in the queue, plus the new PR, and merges them all into this temporary branch.  
> 4. **The Test:** Your CI/CD pipeline runs its automated tests against this temporary integration branch.  
> 5. **The Outcome:**  
   * **If the tests PASS:** GitHub automatically merges this combined state into main and deletes the temporary branch.  
   * **If the tests FAIL:** GitHub automatically discards the temporary throwaway branch, kicks the failing PR out of the queue (alerting the developer), and immediately creates a *new* throwaway branch with the remaining valid PRs to test them.

### **Why This is the Ultimate Evolution of the Strategy**

By using Merge Queues, organizations achieve the exact same benefits of your manual throwaway integration branches, but at machine speed:

> * **The Mainline is Untouchable:** Code only ever reaches main if the *combined* integration of all queued features has been proven to work together.  
> * **Zero Developer Overhead:** Developers do not have to manually cut integration branches, resolve cascading merge conflicts, or coordinate who is deploying what. The platform orchestrates the bundling and testing.  
> * **Painless Backouts:** Just like your manual strategy, if a feature breaks the build in the queue, it is simply discarded from the integration bundle without leaving a trace of dirty commit history on main.

While your team currently relies on manual QA, meaning you must manually create and hold these throwaway branches open for longer periods, migrating to automated tools like GitHub Merge Queues is exactly the path you will take once your automated testing suite matures.

---

*User prompt: What if there is merge conflict of the integration branch in git hub*

Response: When dealing with **Throwaway Integration Branches**, encountering a merge conflict in GitHub introduces a unique architectural dilemma.

Because the integration branch is temporary and disposable, how you handle the conflict determines whether your individual feature branches remain clean or become hopelessly entangled with other developers' code.

Here is the exact workflow and mindset your team should adopt when a GitHub Pull Request into the throwaway integration branch shows a merge conflict.

### **The Core Rule: Keep Feature Branches Pure**

If **Feature B** has a conflict with **Feature A** (which was already merged into the integration branch), **do not merge Feature A into Feature B's branch to resolve it.**

If you do this, you have permanently contaminated Feature B with Feature A's code. If QA subsequently tests the integration branch and decides that Feature A is broken and must be rejected, you can no longer easily deploy Feature B, because Feature B now contains Feature A's broken code.

### **The Standard Resolution Workflow**

When GitHub blocks a Pull Request to the integration branch due to a conflict, you should resolve the conflict **only on the integration branch itself**.

Here are the steps your team should follow:

**1\. Identify the Conflict Source** Look at the GitHub PR to see what files are conflicting. Determine if the conflict is against code from main, or against another feature that was recently merged into the integration branch.

**2\. Update from Main First** If the conflict is simply because the feature branch is out of date with main, the developer should pull main into their feature branch locally, resolve the conflicts, and push. This updates the PR and often resolves the GitHub block.

**3\. Resolve Feature-to-Feature Conflicts on the Integration Branch** If the conflict is between two unreleased features (Feature A and Feature B), the resolution must happen at the point of integration.

> * **Via GitHub UI:** You can use GitHub's web editor ("Resolve conflicts" button) on the PR. This will create a merge commit that resolves the conflict *specifically for this integration branch*.  
> * **Via Local CLI:** The developer checks out the integration branch locally, runs git merge feature-b, resolves the conflicts in their IDE, commits, and pushes the integration branch back to GitHub.

### **The "Recreation Tax" (And Why It Is Worth It)**

Because you resolved the conflict directly on the throwaway integration branch, that resolution is tied exclusively to that branch.

**What happens if the integration branch fails QA and must be deleted?** If you delete the integration branch and create a new one, and you attempt to merge Feature A and Feature B together again, **you will have to resolve the exact same merge conflict a second time.**

This is known as the "recreation tax." It is the known cost of doing business with manual integration branches. However, it is a highly beneficial trade-off because:

> * Your individual feature branches remain 100% isolated and pure.  
> * You maintain the absolute freedom to drop any feature from the release at a moment's notice without having to unpick intertwined code.

### **Pro-Tip for the DevOps Team: Enable git rerere**

If your team finds themselves resolving the exact same conflicts multiple times because integration branches are frequently rebuilt, you can instruct developers to enable a native Git feature called **rerere (Reuse Recorded Resolution)** locally.

Command: git config \--global rerere.enabled true

When this is enabled, Git watches how a developer resolves a specific merge conflict. If that exact same conflict occurs again in the future (e.g., when rebuilding a throwaway integration branch), Git will automatically apply the developer's previous resolution, completely eliminating the "recreation tax."

---

*User prompt: Draw a diagram infochart with details how does integration branch works , I need this to discuss with the team*

Response: Here is a visual flowchart designed specifically for your team presentation, followed by structured talking points. You can use this diagram to clearly illustrate how the Throwaway Integration Branch strategy protects the main codebase while accommodating manual QA.

### **The Throwaway Integration Branch Workflow**

flowchart TD  
    classDef mainBranch fill:\#1f77b4,stroke:\#fff,stroke-width:2px,color:\#fff;  
    classDef featureBranch fill:\#2ca02c,stroke:\#fff,stroke-width:2px,color:\#fff;  
    classDef intBranch fill:\#ff7f0e,stroke:\#fff,stroke-width:2px,color:\#fff;  
    classDef qaNode fill:\#d62728,stroke:\#fff,stroke-width:2px,color:\#fff,shape:diamond;  
    classDef action fill:\#9467bd,stroke:\#fff,stroke-width:2px,color:\#fff;

    Main\[(main Branch\\nAlways Stable)\]:::mainBranch

    subgraph Feature Development  
        F\_A(Feature A\\ne.g., Schema Update):::featureBranch  
        F\_B(Feature B\\ne.g., Shared Library):::featureBranch  
        F\_C(Feature C\\ne.g., Order Routing):::featureBranch  
    end

    Main \--\> F\_A  
    Main \--\> F\_B  
    Main \--\> F\_C

    subgraph First Integration Attempt  
        RC1(Integration Branch 1\\n'rc-v1.0'):::intBranch  
        Main \--\>|1. Cut branch| RC1  
        F\_A \-.-\>|2. Merge| RC1  
        F\_B \-.-\>|2. Merge| RC1  
        F\_C \-.-\>|2. Merge| RC1  
          
        QA1{QA / UAT\\nTesting}:::qaNode  
        RC1 \--\>|3. Deploy to Staging| QA1  
    end

    subgraph Backout & Re-Integration  
        Toss\[4. Feature B Fails\!\\nTHROW AWAY BRANCH\]:::action  
        QA1 \--\>|Bug Found| Toss  
          
        RC2(Integration Branch 2\\n'rc-v1.1'):::intBranch  
        Main \--\>|5. Cut NEW branch| RC2  
          
        F\_A \-.-\>|6. Merge| RC2  
        F\_C \-.-\>|6. Merge| RC2  
          
        QA2{QA / UAT\\nTesting}:::qaNode  
        RC2 \--\>|7. Deploy to Staging| QA2  
    end

    subgraph Production Release  
        MergeMain\[8. Merge to main\\n& Tag Release\]:::action  
        QA2 \--\>|All Clear| MergeMain  
        MergeMain \--\> Main  
    end

    %% Workflow connections  
    Toss \-.-\>|Discard & Rebuild| RC2

### **Key Talking Points for Your Team Presentation**

When presenting this to your 20-person group, focusing on how this strategy manages complex deployments across different domains will help gain buy-in.

**1\. The Goal: Absolute Purity of main** Our primary objective is that main must never contain broken code. Because we rely on manual QA/UAT rather than a fully automated test suite, we cannot merge directly into main. The integration branch acts as our staging ground to compile and verify code before it ever touches production.

**2\. Isolation of Features** Notice in the diagram that when a bug is found in Feature B, **we do not try to untangle it.** When managing deployments, schemas, and shared libraries across multiple teams, attempting to git revert intertwined code often causes cascading failures. By tossing the integration branch, Feature A and Feature C remain 100% pure and unaffected by Feature B's failure.

**3\. The "Recreation Tax" is a Feature, Not a Bug** Rebuilding the integration branch (Step 5\) and re-merging the approved features (Step 6\) might seem like repetitive work. This is the known "recreation tax." However, this minor upfront cost guarantees a completely clean Git history and prevents us from accidentally deploying partial bug fixes.

**4\. Conflict Resolution Protocol** If a merge conflict happens between Feature A and Feature C when building the integration branch, **resolve it on the integration branch only.** Never merge Feature A into Feature C's branch to resolve a conflict. Keeping feature branches completely isolated from one another is what gives us the flexibility to drop any feature at any time.

**5\. The Pathway to Automation** This is our current strategy because it fits our testing reality. As we build out more comprehensive unit and integration testing within our Gradle submodules, we will slowly replace this manual rebuild process with automated CI pipelines, paving the way toward Trunk-Based Development.

To help tailor the next steps for your team's workflow, how frequently are your developers currently deploying these release candidates to the UAT environment for testing (e.g., daily, weekly, or at the end of a sprint)?

---

*User prompt: Create a png file for me to download*

Your PNG file is ready \[file-tag: /mnt/data/throwaway\_integration\_workflow.png\]

This diagram visualizes the exact workflow we discussed, clearly highlighting the separation between the stable main branch, isolated feature development, and the ephemeral nature of the integration branches.

You can use this visual to walk your team through the "Discard & Rebuild" process when a bug is found (like Feature B in the example), showing how it prevents broken code from tangling with healthy features before reaching the production release.

---

