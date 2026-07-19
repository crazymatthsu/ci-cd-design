The **12-Factor Strategy** (or Twelve-Factor App methodology) is a widely used set of best practices created by engineers at Heroku for building modern, scalable, and resilient cloud-native software. It helps developers easily onboard new team members, deploy anywhere, and scale without major architecture overhauls. \[[1](https://www.geeksforgeeks.org/blogs/what-is-twelve-factor-app/), [2](https://www.youtube.com/shorts/CsttKzzbaZs), [3](https://medium.com/the-developers-journal/12-factor-app-principles-explained-ff619d7b7275)\]

The twelve core principles outline exactly how software should be structured and maintained: \[[1](https://www.youtube.com/watch?v=UORHhj6EbII&t=149), [2](https://www.geeksforgeeks.org/blogs/what-is-twelve-factor-app/)\]

1. **Codebase:** Maintain exactly one version-controlled codebase tracked in a repository (like Git), used to deploy across multiple environments (staging, production).  
2. **Dependencies:** Explicitly declare and isolate all libraries and dependencies (using tools like package-lock.json or pip), rather than relying on system-wide installations.  
3. **Config:** Strictly separate configuration (database credentials, API keys, and environment-specific variables) from the actual code, storing them as environment variables instead.  
4. **Backing Services:** Treat backing services (like databases, message queues, and cache systems) as attached resources, meaning they can be swapped or scaled independently without changing the application code.  
5. **Build, Release, Run:** Strictly separate the build (compiling code), release (combining build with config), and run (launching the app execution) stages.  
6. **Processes:** Execute the application as one or more stateless processes. Any data that needs to persist must be stored in a stateful backing service.  
7. **Port Binding:** The application should be completely self-contained and export its services directly via port binding (e.g., via HTTP), rather than relying on an external web server to run.  
8. **Concurrency:** Scale the application out horizontally by adding more individual, identical processes (rather than making a single process larger/heavier).  
9. **Disposability:** Maximize system robustness by ensuring processes can start up almost instantly and shut down gracefully when a stop signal is received.  
10. **Dev/Prod Parity:** Keep development, staging, and production environments as similar as possible to catch deployment or configuration bugs early.  
11. **Logs:** Treat logs as event streams. The application should simply stream its output (stdout) and let the execution environment or specialized logging tools manage routing, archiving, and analysis.  
12. **Admin Processes:** Run administrative and management tasks (like database migrations) as one-off processes in an environment identical to the long-running application processes  
    what does one code base really mean ? one docker image or everything in one git repo

**"One codebase" means everything is tracked in one Git repository, not a single Docker image.** \[[1](https://json23.medium.com/build-a-better-full-stack-dev-environment-a939b204da6)\]

Codebase vs. Docker Image

* **The Codebase is the Source**: It is the single Git repository (or version control repo) where your developers write and track code.  
* **The Docker Image is an Artifact**: It is a package generated *from* that codebase during the **Build** stage of your deployment pipeline. \[[1](https://12factor.net/codebase), [2](https://tms-outsource.com/blog/posts/what-is-a-codebase/), [3](https://devcookies.medium.com/the-12-factor-app-a-complete-guide-for-building-scalable-and-modern-applications-dc0fcae99bcf), [4](https://dockerlabs.collabnix.com/beginners/imagelayers.html), [5](https://www.reddit.com/r/devops/comments/1fy8fyh/docker_image/)\]

The Strict Rules of One Codebase

To truly follow the 12-Factor methodology, your codebase must meet two strict criteria:

* **One Repo, Many Deploys**: You have exactly one Git repository. From this single repository, you deploy to multiple environments (Local, Staging, Production). The code remains identical across all environments; only the **Configuration** changes. \[[1](https://www.useanvil.com/blog/engineering/introduction-twelve-factor-app/), [2](https://medium.com/@rajasaikiranvemula/frontend-design-systems-monolith-vs-multirepo-vs-monorepo-8395b4e4773d), [3](https://oneuptime.com/blog/post/2026-03-13-gitops-flux-vs-traditional-cicd/view), [4](https://datasavvy.me/2022/10/31/the-reason-we-use-only-one-git-repo-for-all-environments-of-an-azure-data-factory-solution/), [5](https://blog.devops.dev/simple-ci-cd-build-a-minimal-pipeline-that-works-for-small-teams-fc86945a56e0)\]  
* **No Shared Code Across Repos**: If multiple applications share the exact same code, you are violating the rule. To fix this, you must extract the shared code into a standalone library, track it in its own separate repository, and include it as a **Dependency** in your apps. \[[1](https://12factor.net/codebase), [2](https://medium.com/@emonemrulhasan35/mono-repo-vs-multi-repo-front-end-architecture-bff291a9b981)\]

Monorepos vs. Polyrepos

Modern development introduces a nuance to this rule depending on your architecture:

* **Microservices (Polyrepo)**: Each microservice has its own dedicated Git repository. In this case, each individual microservice is considered a distinct "12-Factor App" with its own single codebase. \[[1](https://get.assembla.com/blog/choosing-source-code-repository-for-your-project/), [2](https://www.linkedin.com/pulse/thetwelve-factor-app-methodology-ashish-kumar-singal)\]  
* **Monorepos**: If you store multiple services in a single giant Git repository, the 12-Factor methodology views each independent sub-folder/service as its own codebase, provided they can be deployed independently.

If you are setting up a pipeline, tell me:

* Do you have **multiple services** or a **single application**?  
* Are you using a **Monorepo** or **separate Git repos**?

I can show you exactly how to structure your **CI/CD build pipeline** for your specific setup.

you said "No Shared Code Across Repos", does that mean we duplicated codes everywhere ?

**No, you do not duplicate code everywhere; you convert shared code into an external dependency.**

The 12-Factor App methodology strictly forbids code duplication. When the rule says "No Shared Code Across Repos," it means you should never copy-paste code between repositories, nor should multiple distinct apps live in the same repository just to share code.

How to Share Code Correctly (The 12-Factor Way)

Instead of duplicating code or creating messy repository links, you use a **Package Manager** to handle shared code. \[[1](https://www.humansecurity.com/tech-engineering-blog/code-sharing-in-microservices-architecture-1/)\]

1. **Isolate the code**: Move the shared code (like a shared authentication utility or database helper) into its own dedicated Git repository.  
2. **Publish it as a package**: Build and publish that code to a private or public package registry.  
3. **Declare it as a dependency**: Use your language's package manager to install it. \[[1](https://github.com/dagster-io/dagster/discussions/31890), [2](https://medium.com/@subham11/shared-library-reusing-common-code-across-services-9e18264b084e), [3](https://yashmahalwal.medium.com/sharing-code-between-projects-1c35a8df456e), [4](https://docs.dagster.io/examples/best-practices/shared-module), [5](https://www.reddit.com/r/PHP/comments/pjj49c/what_is_the_best_way_to_share_code_between/)\]

Examples by Language

Depending on your tech stack, you pull in that shared repository like any other open-source library:

* **Node.js**: Publish to a private **npm** registry and list it in package.json.  
* **Python**: Publish to a private **PyPI** server and list it in requirements.txt.  
* **Java/Kotlin**: Publish to a **Maven** registry and include it in build.gradle or pom.xml.  
* **Go**: Reference the shared Git repository URL directly in your go.mod file. \[[1](https://learn.microsoft.com/en-us/answers/questions/2338530/how-to-share-code-over-multiple-function-apps-\(pyt)\]

Why This Matters for 12-Factor

This approach honors **Factor II (Dependencies)**. By converting shared code into a versioned package (e.g., my-shared-auth v1.2.0), App A can safely upgrade to v1.3.0 for a new feature without accidentally breaking App B, which is still running safely on v1.2.0.

If you have specific code you need to share right now, let me know:

* **What kind of code** is it? (e.g., database models, utility functions, UI components)  
* **What language** are your apps built in?

I can give you the **exact steps** to package it up cleanly.

tell me more about " App A can safely upgrade to v1.3.0 for a new feature without accidentally breaking App B, which is still running safely on v1.2.0." , how to convince a team to adopt this approach

To convince a team to adopt this approach, you need to reframe it from an "architectural rule" into a **risk mitigation and velocity strategy**. Developers hate breaking things, and managers hate delays. This approach solves both problems. \[[1](https://www.fastruby.io/blog/custom-deprecation-behavior.html)\]

Here is a roadmap of arguments, rebuttals, and strategies to get your team on board.

---

1\. The Core Argument: Decoupling Velocity

When code is shared via direct reference or copy-pasting, **App A and App B are tightly coupled.** If App A needs a change to a shared database model, a developer modifies the shared code.

* **The Old Risk:** App A deploys successfully. Two weeks later, App B deploys for an unrelated feature and crashes in production because the shared code changed out from under it. \[[1](https://forums.docker.com/t/docker-swarm-how-to-detect-automatic-rollback/70402)\]  
* **The 12-Factor Solution:** With versioned packages, App A upgrades the dependency to v1.3.0, tests it, and deploys. App B's repository remains completely untouched, still locked safely to v1.2.0. App B's team can schedule the upgrade to v1.3.0 on their own timeline, when they have time to test it.

**The Pitch:** *"Explicit versioning allows Team A to move at 100 mph without accidentally pulling the rug out from under Team B."*

---

2\. How to Counter the "Too Much Overhead" Objection

The number one pushback you will get from the team is: *"It takes too long to make a change, publish a package, and then bump the version in the main app."*

Counter this by focusing on **automation and local development tooling**:

* **Local Linking:** Show them that they don't have to publish to a registry during daily development. Every modern package manager supports local linking (e.g., npm link, yarn link, Python pip install \-e, or Go replace directives). They can edit the shared code and see changes in their app instantly. \[[1](https://www.facebook.com/AnthropicAI/videos/our-thinking-is-influenced-by-the-space-in-which-it-happens/1315093529944278/)\]  
* **CI/CD Automation:** Propose a pipeline where pushing a tag to the shared repository automatically runs tests, builds the package, and publishes it to your private registry (like GitHub Packages, AWS CodeArtifact, or JFrog Artifactory). The developer just writes code and pushes a Git tag; the machine does the rest. \[[1](https://www.instagram.com/reel/Daxrb2YRQG7/), [2](https://bowtiedraptor.substack.com/p/mlops-13-cicd)\]

---

3\. Use the "Blast Radius" Analogy

Ask the team: *"If our shared authentication logic has a critical bug or requires a breaking change, do we want to break all 10 of our microservices simultaneously, or do we want to fix them one by one?"*

Explicit versioning shrinks the **blast radius** of changes. It turns a catastrophic, company-wide breaking change into a controlled, opt-in upgrade path.

---

4\. Provide a Low-Friction Migration Path

Don't ask the team to rewrite everything overnight. Propose a crawl-walk-run approach:

* **Step 1 (Crawl):** Keep the code in separate repos, but don't set up a private package registry yet. Most package managers allow you to point a dependency directly to a Git URL and a specific commit hash or tag (e.g., "my-shared-lib": "git+https://github.com"). This gives you version pinning immediately with zero infrastructure overhead.  
* **Step 2 (Walk):** Pick **one** highly stable piece of shared code (like a standard logging formatter or a set of custom UI constants) and turn it into a versioned package as a pilot project.  
* **Step 3 (Run):** Once the team sees how painless the pilot project is, roll it out to more volatile shared code (like database clients or API wrappers).

---

To help tailor your pitch, let me know:

* What is your team's **biggest fear** about this? (e.g., maintenance overhead, setting up registries, pipeline complexity)  
* Are you working in a **monorepo** or a **multi-repo (polyrepo)** setup?

I can give you a specific **technical blueprint** or a script to address their exact concerns.

we use IntelliJ to work on multiple git repo, what's the best practices to setup IntelliJ workspace, what's the directory structure it should look like , can I use IntelliJ modules to link multiple repos together , to compile Polyrepo together, make changes together ? what's the Polyrepo management strategy for developers making changes in multiple repos

1\. IntelliJ Workspace Setup & Directory Structure

To work efficiently with multiple repositories in IntelliJ IDEA, you should use the **"One Project, Multiple Content Roots"** approach. Instead of opening each repository in a completely separate IntelliJ window, you open a single "parent" or "workspace" window and attach the other repositories to it.

**Recommended Directory Structure**

Create a dedicated root folder on your machine to act as your workspace. Do not initialize Git at this root folder; it is purely an organizational container.

text  
development-workspace/          \<-- Parent folder (Open this in IntelliJ)  
├── .idea/                      \<-- IntelliJ workspace-wide settings (ignored in global git)  
├── shared-auth-lib/            \<-- Repo 1 (Git clone)  
│   ├── src/  
│   └── pom.xml / build.gradle  \<-- Build file  
├── order-service/              \<-- Repo 2 (Git clone)  
│   ├── src/  
│   └── pom.xml / build.gradle  \<-- Build file  
└── notification-service/       \<-- Repo 3 (Git clone)  
    ├── src/  
    └── pom.xml / build.gradle  \<-- Build file

Use code with caution.

**How to open this in IntelliJ:**

1. Open IntelliJ and click **Open**.  
2. Select the top-level development-workspace/ directory.  
3. IntelliJ will automatically scan the subfolders. If it detects Maven or Gradle files, it will prompt you to import them. If it doesn't, go to **File \-\> New \-\> Module from Existing Sources** and select the build file of each sub-repository. \[[1](https://janikvonrotz.ch/2019/01/11/open-multiple-projects-in-intellij/), [2](https://stackoverflow.com/questions/38095385/resolve-dependencies-from-workspace-projects-in-idea)\]

---

2\. Compiling and Linking Polyrepos with IntelliJ Modules

Yes, you can absolutely use IntelliJ modules to link multiple repositories together, compile them simultaneously, and step-through debug across them. \[[1](https://www.reddit.com/r/IntelliJIDEA/comments/1hutpl/multimodule_project_git_pull_all_at_once/)\]

However, you must configure this correctly so your local changes are picked up instantly without needing to publish to a registry.

**For Maven (Using Maven Reactor / Modules)**

To compile them together, you can create a temporary, local-only pom.xml at the root development-workspace/ directory that defines your repositories as modules:

xml  
\<\!-- development-workspace/pom.xml (Do not commit this to any repo) \--\>  
\<project\>  
    \<modelVersion\>4.0.0\</modelVersion\>  
    \<groupId\>local.workspace\</groupId\>  
    \<artifactId\>workspace-parent\</artifactId\>  
    \<version\>1.0-SNAPSHOT\</version\>  
    \<packaging\>pom\</packaging\>  
      
    \<modules\>  
        \<module\>shared-auth-lib\</module\>  
        \<module\>order-service\</module\>  
        \<module\>notification-service\</module\>  
    \</modules\>  
\</project\>

Use code with caution.

When you import this root pom.xml, IntelliJ and Maven will realize order-service depends on shared-auth-lib. When you hit "Compile" or "Debug", IntelliJ will compile the local code of shared-auth-lib and inject it straight into order-service dynamically.

**For Gradle (Using Composite Builds)**

Gradle handles polyrepos natively using a feature called **Composite Builds**. At your root development-workspace/ directory, create a settings.gradle file: \[[1](https://blog.softwaremill.com/monorepo-with-gradle-b000d7b58eef), [2](https://www.reddit.com/r/java/comments/w8vd05/splitting_software_into_multiple_applications_and/), [3](https://www.petrikainulainen.net/programming/gradle/getting-started-with-gradle-creating-a-multi-project-build/)\]

groovy  
// development-workspace/settings.gradle (Do not commit this)  
rootProject.name \= 'workspace-parent'

includeBuild 'shared-auth-lib'  
includeBuild 'order-service'  
includeBuild 'notification-service'

Use code with caution.

Gradle will automatically **substitute** any binary dependency (like an artifact fetched from a remote server) with your local, live-editing code folder if the group and name match.

---

3\. Polyrepo Management Strategy for Multi-Repo Changes

When a developer needs to make a single feature change that spans across shared-auth-lib and order-service, managing Git across multiple repositories can become overwhelming. Use these native IntelliJ features and strategies to keep it clean:

**Visual Anchors for Multi-Repo Management**

* **Enable Version Control Integration**: Ensure IntelliJ recognizes all Git roots. Go to **Settings \-\> Version Control \-\> Directory Mappings**. Ensure every single sub-folder repo is listed there as Git. \[[1](https://www.reddit.com/r/Jetbrains/comments/1214d7a/how_do_you_efficiently_work_on_multiple_repos_in/), [2](https://youtrack.jetbrains.com/articles/SUPPORT-A-3077/Use-two-different-git-servers-in-a-single-project), [3](https://www.jetbrains.com/help/idea/adding-files-to-version-control.html), [4](https://www.jetbrains.com/help/idea/settings-version-control-directory-mappings.html)\]  
* **Group Branches Together**: In the bottom right corner of IntelliJ, click on the Git Branch name. Turn on **"Execute branch operations on all roots"** (or "Synchronous branch control"). When you create a new feature branch (e.g., feature/oauth-upgrade), IntelliJ will automatically create and checkout that exact branch name across *all* open repositories simultaneously. \[[1](https://www.jetbrains.com/help/idea/manage-branches.html), [2](https://medium.com/@kanth.vallampati/monorepo-branching-strategies-for-team-success-bdb0f69936d3)\]  
* **The Log Window**: The **Git \-\> Log** tab in IntelliJ will show a unified timeline of commits across all repositories, color-coded by repo, so you can track your history chronologically.

**Atomic Changes and Pull Request (PR) Workflow**

1. **Branching**: Use the synchronous branch control mentioned above to create feature/xyz across your repos.  
2. **Coding**: Make changes to shared-auth-lib. Because of the Maven/Gradle setups above, your changes will instantly reflect in order-service when you run it locally.  
3. **Committing**: In the IntelliJ Commit tool window, your files will be grouped by repository. Review and commit them separately.  
4. **Pushing & PRs**: Push the branches. You must open **two separate Pull Requests** (one for the library, one for the service).  
5. **The Golden Rule of Polyrepo Deployment**: Always merge and deploy the downstream dependency (shared-auth-lib) **first**. Once it successfully builds, passes CI, and publishes its version (e.g., v1.3.0), you can officially update the version string in the order-service PR and merge that second. \[[1](https://www.chegg.com/homework-help/questions-and-answers/disaster-recovery-company-using-archaic-repository-management-system-switching-modern-vers-q214855308)\]

To help fine-tune this configuration for your team, let me know:

* Are you primarily using **Maven**, **Gradle**, or **npm/Node**?  
* Do you want instructions on how to use **Git Submodules** or toolings like **Meta** / **Repo** to automate cloning all these repositories at once?

AI can make mistakes, so double-check responses

does this work for mixing maven and gradle projects together

Yes, it works, but you have to change how you link them. \[[1](https://www.cloudrepo.io/articles/maven-repository-vs-gradle-repository)\]

Because Maven and Gradle use completely different build systems, you cannot use a parent pom.xml or a Gradle settings.gradle composite build to link them directly. \[[1](https://www.javacodegeeks.com/gradle-equivalents-of-maven-commands.html), [2](https://devopscube.com/build-java-application-using-gradle/)\]

Instead, you rely entirely on **IntelliJ’s native Module System** to bridge the gap. IntelliJ acts as an interpreter, translating both Maven and Gradle projects into its own universal internal module format.

---

How to Set It Up in IntelliJ

Follow these exact steps to make a Gradle project recognize a Maven project locally:

1. **Open the Workspace:** Open the top-level parent folder (development-workspace/) in IntelliJ. \[[1](https://www.lagomframework.com/documentation/1.6.x/java/IntellijMaven.html)\]  
2. **Import Both Build Files:**  
   * Open the Maven project's pom.xml as a project/module.  
   * Open the Gradle project's build.gradle as a project/module.  
   * You will see both projects appear in your IntelliJ project tree. \[[1](https://www.javacodegeeks.com/2020/05/maven-tutorial-for-selenium-test-automation.html), [2](https://stackoverflow.com/questions/28295933/difference-between-build-gradle-project-and-build-gradle-module), [3](https://stackoverflow.com/questions/27741215/how-to-import-a-gradle-built-module-as-depencency-in-android-studio), [4](https://hl7.github.io/docs/core-libs/multiple-projects)\]  
3. **Manually Link the Modules:**  
   * Go to **File \-\> Project Structure \-\> Modules**.  
   * Select your **Gradle** module from the list.  
   * Click the **Dependencies** tab on the right side.  
   * Click the **\+ (Add)** icon at the bottom or side of the window and select **Module Dependency**.  
   * Choose your **Maven** module from the list and click OK. \[[1](https://anuragbhandari.com/coding-tech/creating-a-java-10-project-in-intellij-idea-with-junit-5-and-gradle-support-1259/), [2](https://www.javacodegeeks.com/intellij-idea-include-external-jar-example.html), [3](https://kdrozd.pl/how-to-create-a-fat-uber-jar/)\]

Now, IntelliJ understands the relationship. When you edit code in the Maven project, the Gradle project immediately sees the changes in the IDE editor, and you can step-through debug across both seamlessly.

---

The Big Catch: Local Command Line vs. IDE

While this setup works perfectly **inside the IntelliJ user interface**, it will fail on the command line (Terminal).

If you open a terminal and run ./gradlew build inside your Gradle project, Gradle has no idea that the Maven project exists on your local hard drive. It will completely ignore IntelliJ's internal module settings and look for the dependency on a remote server, causing a "Dependency Not Found" error. \[[1](https://javarevisited.blogspot.com/2022/07/maven-interview-questions-with-answers.html)\]

How to Fix the Command Line & CI/CD Pipeline

To ensure developers can run builds in their terminal and your CI/CD pipeline doesn't break, you must use your **Local Maven Repository (.m2)** as a middleman.

**1\. Publish Locally From Maven \[[1](https://www.taboola.com/engineering/android-working-on-a-multiple-library-project/)\]**

When making changes to your Maven project, you cannot just save the file. You must install it to your machine's local Maven cache. Run this command in your Maven repository root: \[[1](https://beecrowd.com/blog-posts/apache-maven-2/), [2](https://github.com/LearnLib/learnlib/issues/37)\]

bash  
mvn clean install

Use code with caution.

This compiles the Maven project and saves the .jar package to your local \~/.m2/repository/ folder. \[[1](https://medium.com/javarevisited/the-ultimate-guide-to-maven-why-its-the-backbone-of-enterprise-java-6c0199df7a05)\]

**2\. Tell Gradle to Look in the Local Maven Cache \[[1](https://medium.com/romin-irani/gradle-tutorial-part-2-java-projects-5aaf99368018)\]**

By default, Gradle only looks at remote servers like MavenCentral. You must explicitly tell Gradle to check your local machine's .m2 folder first before checking the internet. \[[1](https://discuss.gradle.org/t/gradle-settings-xml-file-equivalent-from-maven-to-define-a-global-local-repository/11158)\]

Add mavenLocal() to the top of your repositories block in your Gradle project's build.gradle file: \[[1](https://discuss.gradle.org/t/how-to-change-the-order-of-repositories/23586)\]

groovy  
// build.gradle  
repositories {  
    mavenLocal() // \<--- Add this at the very top  
    mavenCentral()  
}

dependencies {  
    // Treat your local Maven project exactly like an external dependency  
    implementation 'com.yourcompany:shared-auth-lib:1.3.0-SNAPSHOT'  
}

Use code with caution.

The Developer Workflow for Mixed Build Systems

When a developer modifies code in the Maven library and wants to run the Gradle application via the terminal, their workflow looks like this:

1. Edit code in the Maven repository.  
2. Run mvn clean install in the Maven repo terminal (to update the local .m2 jar).  
3. Run ./gradlew run or ./gradlew build in the Gradle repo terminal.

*(Note: If they run the application using the **green "Play" button inside IntelliJ**, IntelliJ will bypass the terminal steps and compile both automatically because of the Module Dependency step you configured earlier).*

