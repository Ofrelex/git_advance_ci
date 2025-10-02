# GITHUB ACTIONS AND CI/CD COURSE: ADVANCED CONCEPTS AND BEST PRACTICES
In this final phase, you will dive deep into the sophisticated aspects of GitHub Actions, learning how to craft maintainable workflows, optimize performance, and prioritize security in your automation processes. This module is designed for those who have grasped the basics of GitHub Actions and are now ready to elevate their skills, ensuring their workflows are not only functional but also efficient, secure, and scalable.

---

# Step-by-step process (teach → practice → assess)

## Step 1 — Kickoff: set expectations & environment

1. Explain goals & outcomes for the advanced module.
2. Verify each student has:

   * a GitHub repo (private ok) with at least one sample app, tests and a `build` step.
   * permission to create Environments, Secrets, and Actions in that repo.
   * (Optional) Cloud accounts for labs (or provide mock deployments).
3. Provide a checklist & starter template repo with `/.github/workflows` and `/.github/actions` skeletons.

**Deliverable:** Student forks starter repo and opens a PR (so you can demo PR-based workflows).

---

## Step 2 — Best practices & maintainability (lecture + lab)

**Teach (short lecture):**

1. Clear naming (workflows, jobs, steps).
2. Document workflows with high-level comments and README per reusable workflow.
3. DRY by creating reusable workflows and composite actions.
4. Pin action versions or pin to SHAs in high-security contexts.
5. Keep single responsibility per workflow (e.g., CI vs deploy).

**Lab (practical):**

* Convert repeated steps in two workflows into 1 reusable workflow and 1 composite action.
* Add a README under `.github/workflows/README.md` describing each workflow’s purpose.

**Example: reusable workflow** (`.github/workflows/reusable-build.yml`):

```yaml
name: reusable-build
on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm run build
```

**Caller** (uses the reusable workflow):

```yaml
name: CI
on: [push, pull_request]

jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '18'
```

**Example: composite action** (`.github/actions/cache-node/action.yml`):

```yaml
name: Cache Node Modules
runs:
  using: composite
  steps:
    - name: Cache npm
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-
```

Use it in workflows: `uses: ./.github/actions/cache-node`

---

## Step 3 — Performance optimization (lecture + labs)

**Teach:**

1. Parallel jobs & matrix strategy.
2. `concurrency` to cancel duplicate runs and avoid wasted work.
3. Caching dependencies & build outputs.
4. Docker layer cache when building container images.
5. Artifact reuse between jobs.

**Key YAML patterns**

**Matrix + parallel jobs**

```yaml
strategy:
  matrix:
    node: [16,18,20]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy: ${{ matrix }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm ci && npm test
```

**Caching dependencies (Node.js example)**

```yaml
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

**Concurrency to avoid duplicate runs**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

**Docker build with registry caching (example)**

```yaml
- uses: docker/login-action@v2
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: docker/build-push-action@v4
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=registry,ref=ghcr.io/${{ github.repository }}/cache:latest
    cache-to: type=inline
```

**Lab tasks:**

* Add caching to CI and measure speed before/after.
* Parallelize tests and show time improvement.
* Build a Docker image with layer caching and push to GHCR.

---

## Step 4 — Security & secrets (lecture + labs)

**Teach:**

1. Principle of least privilege for `GITHUB_TOKEN` (use the `permissions` block).
2. Use OIDC (OpenID Connect) to issue short-lived credentials to cloud providers instead of long-lived secrets.
3. Use GitHub Environments for production secrets and required reviewers.
4. Pin third-party actions (prefer SHAs for high-security orgs).
5. Avoid printing secrets or storing them in logs/artifacts.
6. Use Dependabot and code scanning to detect vulnerable dependencies.
7. Use signed commits / signed artifacts (optional advanced).

**Important YAML: minimal permissions & OIDC example**

```yaml
permissions:
  contents: read
  id-token: write    # needed to request OIDC tokens
  packages: write    # only if you push packages

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-west-2
```

> **Why OIDC?** it avoids storing long-lived cloud secrets in GitHub. The runner obtains a short-lived token tied to the workflow run.

**Lab tasks:**

* Replace a `secrets.AWS_ACCESS_KEY_ID` & secret key deploy flow with an OIDC role assumption flow.
* Create an Environment `production` with required reviewers and use environment-scoped secrets.
* Demonstrate what happens when a workflow tries to access a missing secret (fail fast).

---

## Step 5 — Release strategies, rollouts, and rollback

**Teach:**

1. Implement staged deployments: `staging` → canary → production.
2. Blue/green vs canary tradeoffs.
3. Automatic rollback on health checks or alarms.

**Pattern: staged rollout (high level)**

1. Deploy to `staging` environment automatically on `main`.
2. If smoke tests pass, deploy a small canary (e.g., 10%).
3. Monitor (health checks, metrics). If green, promote to 100%. If failing, rollback to previous revision.

**Simplified YAML for a canary step** (pseudo steps — adapt to your infra):

```yaml
- name: Deploy canary
  run: ./scripts/deploy_canary.sh $IMAGE_TAG 10   # deploy 10% traffic

- name: Run smoke tests
  run: ./scripts/run_smoke_tests.sh

- name: Promote canary
  if: success()
  run: ./scripts/promote_canary.sh

- name: Rollback on failure
  if: failure()
  run: ./scripts/rollback.sh
```

**Lab tasks:**

* Implement a scripted canary with health-check step and automatic promote/rollback logic.
* Write tests that intentionally fail to show rollback behavior.

---

## Step 6 — Observability, auditing, and cost controls

**Teach:**

1. Upload artifacts and test reports (`actions/upload-artifact`) for later inspection.
2. Configure retention periods for artifacts and logs.
3. Use `concurrency` and scheduled workflow frequency limits to control cost.
4. Integrate pipeline alerts (Slack, Teams, email) on failure or after deployments.
5. Enable audit logs and GitHub advanced security features where available.

**Artifact upload example**

```yaml
- name: Upload test reports
  uses: actions/upload-artifact@v4
  with:
    name: test-report
    path: reports/
    retention-days: 7
```

**Lab tasks:**

* Save test artifacts and configure retention.
* Connect a simple webhook or Slack notification on failed workflows.

---

## Step 7 — Runner management & scaling

**Teach:**

1. When to use GitHub-hosted vs self-hosted runners.
2. Secure self-hosted runners (isolate, keep them updated).
3. Autoscaling self-hosted runners (hint: use runners in a cloud + autoscale scripts or use GitHub Actions Runner Controller on Kubernetes).

**Lab tasks (optional/advanced):**

* Provision a self-hosted runner (demo) and run a job on it.
* Setup a simple autoscaling policy (cloud provider scripts or k8s operator).

---

## Step 8 — Supply-chain & governance (advanced topics)

**Teach:**

1. Pin actions and keep dependency policies (Dependabot).
2. Use code scanning, secret scanning, and actions security review.
3. Consider artifact signing (cosign / sigstore) for container images and releases.
4. Establish organization policy for actions allowed in organization settings.

**Lab / assignment:**

* Create an org policy (or local repo checklist) listing allowed actions and pinning requirements.
* Run Dependabot and review a suggested upgrade PR.

---

## Step 9 — Testing, debugging & developer experience

**Teach:**

1. Use `ACTIONS_STEP_DEBUG` and runner debug logs (explain how to enable safely).
2. Provide `workflow_dispatch` jobs for manual runs and `workflow_run` for orchestration.
3. Use `workflow_call` to let teams reuse common jobs.
4. Guide students in producing good error messages and failures that are actionable.

**Lab tasks:**

* Add a `workflow_dispatch` manual trigger with inputs for `version` and `env`.
* Practice debugging a failing step and show how to re-run single job.

---

## Step 10 — Capstone & assessment

**Capstone project** (deliverable): For a chosen sample app:

* Implement maintainable workflows (reusable + composite actions).
* Use caching & matrix builds for speed.
* Implement OIDC-based cloud deployment with staging and production environments and required reviewers.
* Implement a canary rollout with automatic rollback.
* Add documentation, a README, and monitoring/notifications.

**Rubric (suggested):**

* Workflow correctness & tests: 30%
* Security (OIDC, least privilege, pinned actions): 25%
* Performance (caching, parallelization): 15%
* Release & rollback strategy: 15%
* Documentation & developer experience: 15%

---

# Quick troubleshooting checklist (share as cheat-sheet)

* Failing to pick up tags? Use `fetch-depth: 0` on checkout.
* Secrets not present? Confirm secret name and scope (repo vs environment).
* Workflow stuck waiting for approval? Check Environment protection rules.
* Job timing out? Split into smaller jobs or increase timeouts where appropriate.
* Large artifact size? Reduce retention or exclude unnecessary files.

---

# Recommended resources & extras (for instructors)

* Provide students links to GitHub Docs for: reusable workflows, OIDC, environments, actions security.
* Provide a checklist for security review (pin SHAs, review action authors, limit `GITHUB_TOKEN` permissions).
* Offer an optional advanced reading: sigstore/cosign for artifact signing and supply-chain security.

---
