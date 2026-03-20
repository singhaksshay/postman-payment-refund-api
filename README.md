# Postman API Onboarding — Working Implementation

**Author:** Akshay Singh  
**Exercise:** Postman Customer Success Engineer — Technical Assessment (Part 1)  
**Scope:** End-to-end onboarding of 3 financial services APIs into Postman using pre-built GitHub Actions

---

## Overview

This implementation onboards three OpenAPI-specified services into Postman workspaces using a GitHub Actions pipeline built on Postman's pre-built composite action suite. Each service lives in its own GitHub repository, mirroring how this tooling operates in real enterprise engagements — one repo per service, one workflow per service, independent Postman workspaces.

**Onboarded Services:**

| Service | Repo | Domain | Environments |
|---|---|---|---|
| Payment Refund API | `singhaksshay/postman-payment-refund-api` | payments | prod, uat, qa, dev |
| Loan Origination API | `singhaksshay/postman-loan-origination-api` | lending | prod, staging, dev |
| Claims Processing API | `singhaksshay/postman-claims-processing-api` | insurance | prod, staging, dev |

---

## How I Built This

### Step 1 — Reading the Action READMEs

Before writing a single line of YAML, I read all three action READMEs carefully:

- `postman-cs/postman-api-onboarding-action` — the composite orchestrator
- `postman-cs/postman-bootstrap-action` — handles Postman-side setup (workspace, spec, collections)
- `postman-cs/postman-repo-sync-action` — handles repo-side sync (environments, exports, commits)

The key insight was that since I'm calling the composite action (`postman-api-onboarding-action`), I don't need to wire bootstrap and repo-sync manually — the composite handles that internally. My job was just to pass the right inputs.

### Step 2 — Setting Up Authentication

The actions require two separate Postman credentials with different scopes:

- **`POSTMAN_API_KEY`** (PMAK): A long-lived key generated from Postman account settings. Used for all standard Postman API operations — creating workspaces, uploading specs, generating collections.
- **`POSTMAN_ACCESS_TOKEN`**: A session-scoped token extracted after running `postman login` via the CLI. Required for internal operations — workspace ↔ repo linking (Bifrost), governance group assignment, and system environment associations. Without this, those steps silently skip rather than fail.

I also generated a **GitHub PAT** (`GH_FALLBACK_TOKEN`) with `repo` and `workflow` scopes. The default `GITHUB_TOKEN` provided by Actions doesn't have sufficient permissions to write workflow files or repository variables — the fallback token covers those operations.

All three secrets were set on each repo using the GitHub CLI:

```bash
gh secret set POSTMAN_API_KEY --repo singhaksshay/<repo>
gh secret set POSTMAN_ACCESS_TOKEN --repo singhaksshay/<repo>
gh secret set GH_FALLBACK_TOKEN --repo singhaksshay/<repo>
```

### Step 3 — Hosting the Spec Files

The action's `spec-url` input expects a URL the GitHub Actions runner can fetch at build time. Since the OpenAPI specs are checked into each repo, I used raw GitHub URLs:

```
https://raw.githubusercontent.com/singhaksshay/<repo>/main/specs/<spec-file>.yaml
```

I verified each URL returned `HTTP 200` before wiring it into the workflow.

### Step 4 — Writing the Workflow

I used `generate-ci-workflow: false` on all three services. This prevents the action from generating and committing an additional CI workflow file, which would conflict with the onboarding workflow we're already managing. For a fresh onboarding, you'd set this to `true` to let the action generate a standard CI pipeline.

The workflow triggers on:
- `workflow_dispatch` — manual trigger for reruns and demos
- `push` to the spec file or workflow file — automatically re-onboards when the spec changes

### Step 5 — Verifying End-to-End

Each run was verified by:
1. Watching the workflow complete via `gh run watch`
2. Running `git pull` to confirm artifacts were committed back by the action
3. Checking the `postman/` directory structure for all three collection types

---

## Workflow Architecture

```
GitHub Push / workflow_dispatch
        │
        ▼
postman-api-onboarding-action@v0 (composite)
        │
        ├── postman-bootstrap-action@v0
        │       ├── Create or reuse Postman workspace
        │       ├── Upload/update spec in Spec Hub
        │       ├── Lint spec with Postman CLI
        │       ├── Generate [Baseline], [Smoke], [Contract] collections
        │       └── Persist workspace/spec/collection IDs as repo variables
        │
        └── postman-repo-sync-action@v0
                ├── Create/update environments from runtime URLs
                ├── Associate environments to system environments (Bifrost)
                ├── Create mock server + smoke monitor
                ├── Export collections → postman/collections/
                ├── Export environments → postman/environments/
                ├── Link workspace to GitHub repo (Bifrost)
                └── Commit and push all artifacts back to branch
```

---

## What's Universal vs. What Changes Per Service

### Universal (same across all three services)

- The composite action reference: `postman-cs/postman-api-onboarding-action@v0`
- The required secrets: `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GH_FALLBACK_TOKEN`
- The permissions block: `actions: write`, `contents: write`
- The `generate-ci-workflow: false` flag
- The raw GitHub URL pattern for `spec-url`
- The output structure: `postman/collections/`, `postman/environments/`, `.postman/`
- The three collection types generated: Baseline, Smoke, Contract
- Rerun safety — the action reads stored repo variables to avoid creating duplicate Postman assets

### Changes Per Service

| Input | Payment Refund | Loan Origination | Claims Processing |
|---|---|---|---|
| `project-name` | `payment-refund-api` | `loan-origination-api` | `claims-processing-api` |
| `domain` | `payments` | `lending` | `insurance` |
| `domain-code` | `PAY` | `LND` | `INS` |
| `spec-url` | payment spec URL | loan spec URL | claims spec URL |
| `environments-json` | prod, uat, qa, dev | prod, staging, dev | prod, staging, dev |
| `env-runtime-urls-json` | 4 environment URLs | 3 environment URLs | 3 environment URLs |

The environments differed because each spec defined different server environments. Payments had a UAT and QA environment in addition to prod and dev — I mapped those directly from the OpenAPI `servers` block rather than inventing environment names.

---

## Spec Differences Across Services

Understanding the spec characteristics informed the workflow decisions and surfaced what the consulting adaptation would need to address.

### Payment Refund API
- **Auth:** OAuth 2.0 + JWT (dual scheme)
- **Environments:** 4 (prod, uat, qa, dev) — most granular of the three
- **Complexity:** Highest — rich request/response examples, HATEOAS links, partial refund logic, pagination
- **Lint findings:** Schema properties using inline definitions instead of `$ref` — minor, doesn't affect collection generation

### Loan Origination API
- **Auth:** JWT / mTLS (service-to-service uses mutual TLS)
- **Environments:** 3 (prod, staging, dev)
- **Notable:** Multipart file upload endpoint (`/documents`), async underwriting trigger, business rules around applicant eligibility
- **Lint findings:** Missing 5xx responses on several operations, schema `$ref` recommendations — slightly lower spec maturity than Payments

### Claims Processing API
- **Auth:** Dual scheme — OAuth 2.0 (external) + API Key via `X-API-Key` header (internal)
- **Environments:** 3 (prod, staging, dev)
- **Infrastructure note:** Runs on mixed Lambda/ECS; uses **GitLab CI** for deployments — not GitHub Actions
- **Lint findings:** Same pattern as Loans — missing 5xx responses, schema `$ref` recommendations
- **Key consulting implication:** This service was onboarded using GitHub Actions for this exercise, but in a real engagement the CI/CD trigger would need to be adapted for GitLab CI. The Postman workspace, spec, and collections are identical — only the pipeline trigger changes.

---

## Run Instructions

### Prerequisites

- Postman Enterprise account with API key (`PMAK-...`)
- Postman CLI installed and authenticated (`postman login`)
- GitHub CLI installed and authenticated (`gh auth login`)
- GitHub PAT with `repo` and `workflow` scopes

### Initial Setup (per repo)

```bash
# Clone the repo
gh repo clone singhaksshay/<repo-name>
cd <repo-name>

# Set secrets
gh secret set POSTMAN_API_KEY --repo singhaksshay/<repo-name>
gh secret set POSTMAN_ACCESS_TOKEN --repo singhaksshay/<repo-name>
gh secret set GH_FALLBACK_TOKEN --repo singhaksshay/<repo-name>
```

### Triggering the Workflow

**Automatic:** Push a change to the spec file or workflow file.

**Manual:**
```bash
gh workflow run onboard-api.yml --repo singhaksshay/<repo-name>
```

**Watch the run:**
```bash
gh run watch --repo singhaksshay/<repo-name>
```

### Rerun Behavior

On rerun, the action reads stored GitHub repository variables (`POSTMAN_WORKSPACE_ID`, `POSTMAN_SPEC_UID`, collection UIDs) and reuses existing Postman assets rather than creating duplicates. The spec is re-uploaded from the source URL on every run, keeping Spec Hub in sync with the repo.

### Validating the Output

After a successful run, pull the committed artifacts:

```bash
git pull
ls postman/collections/   # Should show [Baseline], [Smoke], [Contract] folders
ls postman/environments/  # Should show one JSON file per environment
cat .postman/config.json  # Workspace ID, spec ID, collection IDs
```

---

## Warnings Encountered (Not Failures)

### 1. Requester invite failed
```
Failed to invite requester: 400 Bad Request — Only one role is supported for user
```
Expected. The `requester-email` is already the workspace owner — Postman rejected the duplicate role assignment. This would succeed in a real customer engagement where the requester is a different user than the API key owner.

### 2. Node.js 20 deprecation
The actions currently run on Node.js 20, which GitHub Actions will deprecate in June 2026. Not a functional issue today — worth tracking for when Postman releases updated action versions.

### 3. Spec lint warnings
All three specs produced lint warnings from the Postman CLI:
- Schema properties using inline definitions instead of `$ref` references
- Missing 5xx responses on some operations (Loans and Claims)

These are spec quality issues, not blocking errors. In a real engagement, I'd flag these to the API team as quick wins to improve spec maturity before or alongside the onboarding.

---

## What the Customer's Ops/Platform Team Needs to Configure

These are the items that couldn't be completed in this exercise because they require the customer's actual infrastructure access:

### Required from the Customer

| Item | Why It's Needed | Who Provides It |
|---|---|---|
| Postman Enterprise license | Required for Spec Hub, governance, workspace linking | IT / Procurement |
| Postman workspace admin user IDs | To grant team access during onboarding (`workspace-admin-user-ids`) | Postman admin |
| System environment IDs (`system-env-map-json`) | Maps environment slugs to Postman system environments for governance | Postman admin |
| Governance mapping (`governance-mapping-json`) | Maps domain names to governance group names | Postman admin |
| Real runtime base URLs | Production/staging/dev URLs for environment configuration | Platform/DevOps team |
| CI/CD pipeline access | To add the onboarding workflow to existing pipelines | Platform/DevOps team |

### GitLab CI Adaptation (Claims Processing)
For services running GitLab CI (like Claims Processing), the platform team needs to:
- Add `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, and `GH_FALLBACK_TOKEN` as GitLab CI/CD variables
- Adapt the workflow YAML to GitLab CI syntax (`.gitlab-ci.yml`)
- Note: The composite action is GitHub Actions-specific — GitLab would need to call the underlying Postman API directly or use the Postman CLI in a GitLab runner

### mTLS Services (Loan Origination)
For services using mutual TLS authentication:
- The client certificate and key need to be provided as secrets
- Postman environments need to be configured with the mTLS credentials
- This is not handled automatically by the onboarding action — it requires manual Postman environment configuration post-onboarding

---

## Trade-offs and Design Decisions

**One repo per service vs. monorepo:** I followed the recommended pattern of one repo per service. This mirrors real enterprise engagements and gives each team ownership of their own onboarding pipeline. A monorepo approach would complicate the workflow trigger logic and repo variable namespacing.

**Using the composite action vs. calling bootstrap and repo-sync directly:** The composite action is the right choice for standard onboarding. Calling the lower-level actions directly only makes sense when you need to decouple the bootstrap and sync phases — for example, if bootstrap runs in one pipeline and sync runs in another.

**`generate-ci-workflow: false`:** I set this to false because the repos already have an onboarding workflow and we don't want the action generating a second one. In a greenfield service with no existing CI, you'd set this to `true` to get a generated CI workflow committed automatically.

**Session-scoped access token:** The `POSTMAN_ACCESS_TOKEN` expires and requires manual renewal via `postman login`. This is an open-alpha limitation that Postman plans to address before GA. For a production rollout across 50 services, this would need an automated refresh mechanism or organizational-level token management.

---

## Repository Structure (Post-Onboarding)

```
<repo>/
├── .github/
│   └── workflows/
│       └── onboard-api.yml          # Onboarding workflow
├── .postman/
│   ├── config.json                  # Workspace ID, spec ID, collection IDs
│   └── resources.yaml               # Resource mapping for reruns
├── postman/
│   ├── collections/
│   │   ├── [Baseline] <service>/    # All endpoints, no test assertions
│   │   ├── [Smoke] <service>/       # Happy path tests with secret resolution
│   │   └── [Contract] <service>/    # Full contract validation tests
│   └── environments/
│       ├── prod.postman_environment.json
│       ├── staging.postman_environment.json
│       └── dev.postman_environment.json
├── specs/
│   └── <service>-openapi.yaml       # Source of truth OpenAPI spec
└── README.md
```

---

## References

- [Postman API Onboarding Action](https://github.com/postman-cs/postman-api-onboarding-action)
- [Postman Bootstrap Action](https://github.com/postman-cs/postman-bootstrap-action)
- [Postman Repo Sync Action](https://github.com/postman-cs/postman-repo-sync-action)
- [Postman API Authentication](https://learning.postman.com/docs/developer/postman-api/authentication/)
- [Postman Spec Hub](https://learning.postman.com/docs/designing-and-developing-your-api/managing-apis/)
- [Postman CLI](https://learning.postman.com/docs/postman-cli/postman-cli-overview/)
