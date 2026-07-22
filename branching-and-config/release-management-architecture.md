# Release Management Architecture — Global Java Platform

Design for a global Java estate (US / Hong Kong / London; equities, fixed income, swaps; QA / UAT / Prod) migrating from a single monorepo-with-config + cherry-pick model to a modern build-once / promote-everywhere pipeline.

---

## 1. The root cause of the cherry-picking pain

Cherry-picking is a **symptom**, not the disease. It happens because today one Git branch encodes three unrelated dimensions at once:

1. **What the code is** (a version of business logic)
2. **Where it runs** (QA vs UAT vs Prod, US vs HK vs London)
3. **How it's configured** (env/region-specific values)

When environments and regions live on branches, every fix must be manually replayed (cherry-picked) onto every branch that represents an environment. The fix is to make each dimension live where it belongs:

| Dimension | Where it should live |
|---|---|
| Code version | Git history + **immutable versioned artifact** in an artifact repository |
| Environment/region | **Directories** in a config repo + a deployment manifest — *never branches* |
| Configuration values | Config repo (trunk-only), layered overrides |

Golden rule: **an environment is a place you promote an artifact to, not a branch you merge code into.**

---

## 2. Branching strategy for the Java (binary) repository

### Recommendation: Trunk-Based Development with short-lived release branches

(See `git-branching-trunk-based.png` in this folder.)

```
main ──●──●──●──●──●──●──●──●──●──●──►
            \              \
             release/25.7   release/25.8      ← cut per release train
                 ●              ●
                 │tag 25.7.0    │tag 25.8.0
                 ●  (hotfix)    │
                 │tag 25.7.1    │
```

Rules:

- **`main` is the only long-lived branch.** All work lands on `main` via short-lived feature branches (days, not weeks) and pull requests with mandatory CI + review.
- **Release branches** (`release/25.8`) are cut from `main` on a fixed cadence (a *release train*, e.g. every 2–4 weeks). They exist only for stabilization; only critical fixes go there.
- **Hotfixes**: fix on `main` first, then cherry-pick *one commit to one release branch*. This is the **only** sanctioned cherry-pick in the whole system, it flows in one direction (main → release), and it disappears entirely once you trust the train cadence enough to move to pure tags.
- **Tags, not branches, define what ships.** Every build that could reach Prod is a tag (`25.8.0`, `25.8.1`) producing an immutable artifact.
- **No `qa`, `uat`, `prod`, `hk`, `london` branches. Ever.** Promotion between environments is an artifact-repository/manifest operation (§5), not a Git operation.
- **Unfinished features** are handled with **feature flags** (config-driven), not long-lived branches. A flag in the config repo turns a flow on in QA-HK while it stays off in Prod-US — same binary everywhere. This is also how you decouple *business go-live dates* (which differ by region) from *deployment dates*.

### Variant: when the team has weak/no automated tests (human QA/UAT only)

Pure trunk-based development assumes automated tests are the merge gate. Without them, `main` is *not* "always releasable" — it's "compiles and probably works." Human QA needs a **frozen, stable target** to test against, and that changes the model in two ways:

**1. Release branches become longer-lived stabilization branches (this is fine).**

```
main ──●──●──●──●──●──●──●──●──●──●──●──►
            \                   ▲    \
             release/25.8 ──●───●     release/25.9
                 │RC1  QA   │RC2 \merge back
                 │          │UAT  ●──tag 25.8.0 → Prod
```

- Cut `release/25.8` from `main` on the train date → build `25.8.0-rc.1` → humans test that exact artifact in QA, then UAT.
- The branch lives for the whole manual test cycle (2–4 weeks is normal). `main` keeps moving for the next train.
- **At most 1–2 active release branches** (the one in QA/UAT + at most one in prod-hotfix support). More than that and you've recreated the merge matrix you're escaping.

**2. Fix direction reverses: fix on the release branch first, merge back to `main`.**

With strong automation, you fix on `main` and cherry-pick to release. With human-only testing, that's backwards — by the time QA finds a bug, `main` contains days of *unverified* new work, and you must not let it ride along into the tested release. So:

- QA/UAT bug → fix committed **on `release/25.8`** → new RC → humans retest only that area.
- Periodically (and at release end) **merge — not cherry-pick — `release/25.8` back into `main`**. A merge carries all fixes at once, tracks ancestry, and can't silently miss one; the chronic cherry-picking disappears even in this variant.

**Everything else in this document stands unchanged.** Weak automation makes build-once *more* critical, not less: manual regression is so expensive that you can only afford to run it once per release — so the tarball humans signed off in UAT must be byte-identical in Prod-US/HK/LDN. Environments still must not be branches; config still lives in the config repo; promotion is still a manifest PR. And keep feature flags **few** — every flag doubles the states humans must test.

**What NOT to do:** don't adopt full Git Flow (`develop` + `main` + release + hotfix). The `develop`/`main` split adds a permanent extra merge for zero benefit once releases are tagged artifacts — with no automation, every extra long-lived branch is another human-tested merge to get wrong.

**This is transitional.** Each release cycle, add characterization/smoke tests for whatever bit QA in that cycle (the highest-value tests to write first). As the automated suite grows, the stabilization window shrinks from 4 weeks toward days, and the model converges on standard trunk + short release branches with no process change — only the branch lifetime changes.

### Feature back-out: throwaway integration branches + ephemeral environments

Problem: 5 features are candidates for a release; QA tests them together; at the last minute feature #3 must be pulled. How do you remove it cleanly?

**Why not `git revert` of the merge commit:** reverting a merge (`git revert -m 1`) removes the *content* but leaves the merge in *history*. When feature #3 is later fixed and re-merged, Git believes its commits are already present and silently merges **nothing** — you must "revert the revert" first. This bites almost everyone eventually. Rebuilding the integration branch from scratch avoids the entire problem.

**Throwaway integration branch rules (dev-1 / dev-2 pattern):**

1. **The integration branch is a build artifact, not a source of truth.** It is *generated* by merging `main` + a candidate list; it is **never merged into anything**, nobody branches from it, nobody commits to it directly. Delete/recreate at will.
2. **Drive it from a manifest, not by hand:**
   ```yaml
   # release-candidates/25.8.yaml  (lives on main, PR-controlled)
   integration: dev-3            # bump on every rebuild
   base: main@<sha>              # pin the base for reproducibility
   features:
     - feature/eq-report-mifid
     - feature/mo-booking-amend
     - feature/swaps-csa-calc
     - feature/fi-curve-bootstrap
     # - feature/eq-alloc-rework   ← backed out: comment out, bump dev-N, CI rebuilds
   ```
   A CI job checks out `base`, merges each listed branch in order, pushes `dev-N`, builds the RC tarball. **Back-out = delete one line + rerun.** The manifest's Git history is the audit record of what was in/out and when.
3. **Bump the number (`dev-1` → `dev-2`), don't force-push the same name** — RC artifacts and QA sign-offs stay traceable to an immutable branch/SHA.
4. **Conflicts:** require feature branches to stay rebased on `main` (each feature owner's job); for cross-feature conflicts, enable `git rerere` on the CI merge job so a resolution recorded once replays on every rebuild. If a conflict needs real work, fix it *in the feature branch*, not in dev-N.
5. **Fixes during QA go to the feature branch**, then rebuild dev-N+1. Never patch dev-N directly (it's disposable — the fix would be lost).
6. **Ship path:** after QA signs off dev-3, merge the four accepted feature branches to `main` (or fast-forward main to dev-3 if main hasn't moved), tag, build the release artifact. Then verify `git diff dev-3 <release-tag>` is **empty** — if the trees are identical, the release binary is byte-for-byte what QA tested and the sign-off carries over without retesting.

**Ephemeral (per-feature) environments — the complementary strategy:** spin up a short-lived environment *per feature branch* so each feature gets functional sign-off **in isolation, before** it ever enters the integration manifest. This shrinks the back-out problem instead of solving it faster: features are pulled *before* integration, and the integrated dev-N cycle only has to catch cross-feature interactions.
- With tarballs on-prem this needs: a pool of VMs/ports, per-env DB schema or anonymized data subset, and the config-repo layering (an `overlays/ephemeral/` layer templated per instance). Doable but operationally heavy.
- With containers/K8s it's nearly free: namespace per feature branch, Argo CD ApplicationSet spins it up on branch creation and tears it down on merge/TTL. This is a strong (and standard) payoff of the containerization path in §7.

**Recommended combination for this team:** throwaway integration branches **now** (pure Git process, works with tarballs and human QA today); add per-feature ephemeral environments when containerized. End state: ephemeral env answers "does this feature work?", dev-N answers "do these features work *together*?" — and either question failing has a one-line back-out.

### Why not Git Flow

Git Flow (`develop` + `release` + `hotfix` + environment branches — see `git-branching-git-flow.png`) is what most enterprises drifted into and is exactly what generates chronic cherry-picking and "which branch is Prod-HK on?" confusion. Its extra long-lived branches solve a problem (no CI, infrequent integration) you're eliminating.

### Handling multiple business flows (equities / FI / swaps)

Keep the single Java repo **as a monorepo of independently deployable modules** rather than splitting repos immediately:

```
trading-platform/                  (one Git repo, one Gradle/Maven multi-module build)
├── libs/                          shared domain + infrastructure libraries
│   ├── core-domain/
│   └── messaging/
├── services/
│   ├── mo-booking/                middle-office booking
│   ├── equities-trade-report/
│   ├── fi-trade-report/
│   └── swaps-lifecycle/
└── build.gradle / pom.xml
```

- Each `services/*` module produces **its own versioned artifact** and has **its own deploy cadence** — equities can release weekly while swaps releases monthly, from the same repo.
- CI builds/tests only affected modules per commit (Gradle build cache, or path-filtered pipelines).
- CODEOWNERS maps `services/equities-*` to the equities team, etc., so global teams don't trample each other.
- Split a module into its own repo **later, only if** a domain team needs fully independent tooling/cadence and the shared-library churn is low. Splitting too early turns compile-time errors into cross-repo version-hell.

---

## 3. The configuration repository

### One repo, one branch, directories for everything

Your proposal is right: **`main` only**. HEAD of `main` = the desired configuration of the entire world. Any historical commit = a full world snapshot (perfect audit trail — regulators love this).

```
platform-config/                       (trunk-only)
├── global/
│   └── defaults.yaml                  values shared by every service everywhere
├── services/
│   └── equities-trade-report/
│       ├── base.yaml                  service defaults (all envs)
│       └── overlays/
│           ├── qa/
│           │   ├── us.yaml
│           │   ├── hk.yaml
│           │   └── ldn.yaml
│           ├── uat/…
│           └── prod/
│               ├── us.yaml            ← ONLY prod-us deltas live here
│               ├── hk.yaml
│               └── ldn.yaml
├── flags/                             feature flags per env/region
└── deploy/                            ← the release manifests (§5)
    └── equities-trade-report/
        ├── qa-us.yaml                 { version: 25.8.0, configRef: <sha> }
        ├── uat-us.yaml
        └── prod-us.yaml
```

Principles:

- **Layered overrides, minimal deltas.** Effective config = `global/defaults` ⊕ `service/base` ⊕ `overlays/<env>/<region>`. Overlay files contain *only differences* (hostnames, DB endpoints, market-calendar, throttles). If QA and Prod files are 95% identical copies, config drift and cherry-picking come back through the side door.
- **A config change is a PR to `main`**, reviewed and CI-validated (schema check, "render effective config for all 9 env×region combos and diff"). Changing a Prod DB endpoint no longer touches the Java repo at all — no code release needed.
- **Protected paths**: require senior/prod-approver review on `overlays/prod/**` and `deploy/**prod**` via CODEOWNERS. That *is* your change-approval workflow, with Git as the audit log.
- **No secrets in Git — even encrypted is second-best.** Config repo holds *references* (`db.password: vault:secret/eq-tr/prod-us#db`); actual secrets live in HashiCorp Vault on-prem (maps 1:1 to AWS Secrets Manager later). If you must keep secrets in-repo temporarily, use SOPS with age/KMS.
- Spring Boot fits this natively: ship the layered YAMLs as external config (`spring.config.import` / `--spring.config.additional-location`), or run Spring Cloud Config Server pointed at this repo. Prefer plain files rendered at deploy time — fewer runtime dependencies, and it translates directly to K8s ConfigMaps later.

---

## 4. Build once, promote everywhere

The pipeline's spine, valid for tarballs today and containers tomorrow:

1. **CI builds one environment-agnostic artifact per service per version** — compiled classes + dependency jars, **zero config inside**. `equities-trade-report-25.8.0.tar.gz`, published to **Artifactory/Nexus**, immutable, never rebuilt.
2. **The same bytes** are deployed to QA, then UAT, then Prod, then every region. What changes per target is only the rendered config placed next to it at deploy time.
3. **Promotion is metadata, not compilation.** Moving 25.8.0 from UAT to Prod = updating one line in a manifest file (§5) + Artifactory promotion (repo move `libs-uat-local` → `libs-prod-local`). If you rebuild for Prod, you're testing one binary and shipping another.

Deploy-time composition (tarball world):

```
deploy job:
  1. pull  equities-trade-report-25.8.0.tar.gz        (Artifactory)
  2. checkout platform-config @ configRef sha          (Git)
  3. render effective config for env=prod region=hk    (merge layers)
  4. fetch secrets                                     (Vault)
  5. assemble runtime dir:  app/ + config/ + secrets
  6. stop / swap symlink / start  + health check       (Ansible)
```

---

## 5. Release manifests — GitOps before Kubernetes

The `deploy/` directory in the config repo declares **what runs where**:

```yaml
# deploy/equities-trade-report/prod-hk.yaml
service: equities-trade-report
version: 25.8.0            # artifact in Artifactory
configRef: 9f3ac21         # platform-config commit to render config from
flagsRef: default
```

- **Deploy = PR that edits this file.** Merge triggers (or authorizes) the deploy job for exactly that env/region.
- **Rollback = `git revert`** of that PR. No rebuild, no cherry-pick, seconds to execute, fully audited.
- **"What is running in Prod-HK right now?"** = `git show main:deploy/.../prod-hk.yaml`. One command, always true.
- **Follow-the-sun rollouts**: promote 25.8.0 to `prod-hk.yaml` during HK's window, watch it through the HK trading day, then `prod-ldn.yaml`, then `prod-us.yaml`. Regional stagger is three tiny PRs, not three branches.

This is exactly the Argo CD model — you're just executing it with Ansible/Jenkins until EKS arrives, so **the migration to Kubernetes changes the executor, not the process**.

---

## 6. CI/CD tooling

### Recommended stack

| Layer | On-prem today | AWS tomorrow | Notes |
|---|---|---|---|
| Source control | GitHub Enterprise / GitLab / Bitbucket DC | same | Whatever you have; branch protection + CODEOWNERS are the hard requirements |
| CI (build/test) | **GitHub Actions self-hosted runners** or **GitLab CI**; Jenkins acceptable if entrenched | identical — runners/agents in AWS | Pipeline-as-code in the repo, path-filtered per module |
| Artifact repository | **JFrog Artifactory** (or Nexus) | same instance or Artifactory SaaS | Hosts tarballs *and* later Docker images and Helm charts — one promotion model |
| Secrets | **HashiCorp Vault** | AWS Secrets Manager / KMS (or keep Vault) | Config repo stores references only |
| CD executor | **Ansible** (or Jenkins deploy jobs) driven by manifest merges | EC2: CodeDeploy/Ansible · EKS: **Argo CD** | The only layer that changes per platform |
| Deploy state | `platform-config` repo `deploy/` dir | same repo becomes Argo CD's source of truth | Unchanged across the migration |

If starting clean: **GitLab CI or GitHub Actions + Artifactory + Vault + Ansible now, + Argo CD when EKS lands.** If Jenkins is entrenched, keep it for CI but move deploy authority into the manifest repo — Jenkins becomes a dumb executor of Git-declared state, and swapping it out later is painless.

### Pipeline per service module

```
PR to main:        compile → unit tests → integration tests → config-render check
merge to main:     build artifact 25.8.0-rc.N → publish to libs-snapshot → auto-deploy QA → smoke tests
cut release/tag:   build 25.8.0 → publish libs-release → (manifest PR) → UAT deploy → automated regression
prod:              manifest PR w/ prod approvers → Artifactory promote → deploy HK → LDN → US → health gates
```

Gates worth automating from day one: unit+integration on PR, effective-config schema validation for all env×region combos, smoke test after every deploy, and a diff report ("25.7.4 → 25.8.0 contains these 37 commits / these config deltas") attached to the UAT/Prod approval PR.

---

## 7. Tarball today → containers tomorrow

The tarball is not your problem — config *inside* the tarball is. Fix that first (§4) and the container migration becomes trivial, because a Docker image is just a tarball with better tooling:

| Step | Change | Effort |
|---|---|---|
| 1. Now | Strip config out of the tarball; render at deploy time; adopt manifests | Process change, no infra |
| 2. Now+ | CI also builds a Docker image from the *same* jars (Jib or Spring Boot buildpacks — no Dockerfile needed, no Docker daemon on CI) and pushes to Artifactory. Nobody deploys it yet; you're building the muscle | Small |
| 3. EC2 phase | Deploy the image via compose/systemd on EC2, config mounted from rendered files / SSM | Moderate |
| 4. EKS phase | Config overlays become **Kustomize overlays** (`overlays/prod/hk` maps 1:1 to your existing directory scheme → ConfigMaps); manifests in `deploy/` become the Argo CD app-of-apps; secrets via External Secrets Operator → Vault/ASM | The process you already run, new executor |

Key invariant at every step: **image = code only, config injected at deploy, promotion = retag/manifest edit, never rebuild.**

---

## 8. Operating model summary

- **Release train** per domain (equities weekly, swaps monthly, …) cut from `main`; hotfix = fix on `main`, single cherry-pick to the live release branch — the only cherry-pick that remains.
- **Code change** → PR to Java repo. **Config change** → PR to config repo. **Deploy/rollback/promotion** → PR to `deploy/` manifests. Three distinct, small, auditable motions that never require touching the others.
- **Environments and regions are directories + manifests**, never branches.
- **One artifact version tested in QA is byte-identical in Prod-US, Prod-HK, Prod-LDN.**
- Regional go-live differences are **feature flags + staggered manifest PRs**, not code divergence.
- Git history of the config repo = complete, regulator-ready record of every config and deployment change in the firm.

### Migration order (each step pays off on its own)

1. Extract config from the Java repo into `platform-config` (trunk-only, layered dirs). Delete config from tarballs; render at deploy.
2. Stand up Artifactory promotion + immutable versioned tarballs; kill environment branches; adopt trunk + release-train branching.
3. Introduce `deploy/` manifests; make Ansible/Jenkins deploy only what manifests declare. Rollback = revert.
4. Add Jib image builds in parallel (unused in prod yet).
5. AWS: EC2 first with the same manifests, then EKS + Argo CD + Kustomize, reusing the same config repo structure.
