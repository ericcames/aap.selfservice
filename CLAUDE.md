# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Does

Automates the installation and configuration of the **AAP Self-Service Automation Portal** on an RHDP-provisioned OpenShift cluster. The portal is built on Red Hat Developer Hub (RHDH) and deployed via the `redhat-rhaap-portal` Helm chart from `charts.openshift.io`. It syncs Organizations, Users, Teams, and Job Templates from AAP Controller and gives end users a point-and-click interface to run automation.

This repo is run **after** `aap.as.code` has bootstrapped AAP and loaded demo config via CaC.

## Required CLIs and Runtime Deps

- `oc` — must be logged in to the OCP cluster before running any playbook
- `helm` — must be available locally
- Collections: `ansible.platform` (AAP API), `kubernetes.core` (OCP/Helm)

Install collections once (shared across all repos). If the Hub token is already in `~/.ansible/ansible.cfg`:
```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg ansible-galaxy collection install -r collections/requirements.yml
```
Otherwise pass the token inline as `ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_TOKEN=<token>`. Collections install to `~/.ansible/collections/`.

## RHDP Environment

This repo targets the **Ansible Product Demo** catalog item on the Red Hat Demo Platform (RHDP). That catalog item provisions:

| Component | Notes |
|-----------|-------|
| AAP 2.6 (Controller, Hub, EDA, Gateway) | Admin credentials on RHDP Services page |
| OpenShift 4.x | API URL and token on RHDP Services page |
| **Bastion host** | SSH access pre-configured with OCP CLI and common utilities |

The bastion host is always available with this catalog item. When starting a new session, ask the user for:
1. AAP URL and password (from RHDP Services page)
2. OCP API URL and token (from RHDP Services page)

The bastion is **not required** — bootstrap playbooks run locally via `connection: local` and talk to AAP and OCP over HTTPS directly. Verified 2026-04-30: full bootstrap completed from a laptop without bastion access. Only ask for bastion credentials if the user cannot reach the OCP API directly (firewall/VPN issues) or needs interactive cluster troubleshooting.

Credentials go in `docs/dev-environment.md` (gitignored). Never store them in committed files.

## Inventory Pattern

Each RHDP environment gets its own named inventory under `inventories/rhdp-<customer>-<demo>/group_vars/all.yml`. Copy `inventories/rhdp-sample-demo/` as a starting point. Sensitive values are **never stored in the inventory** — they are resolved at runtime from environment variables:

| Env Var | Purpose |
|---------|---------|
| `CONTROLLER_HOST` | AAP Controller URL |
| `CONTROLLER_USERNAME` | AAP admin username (default: `admin`) |
| `CONTROLLER_PASSWORD` | AAP admin password |
| `OCP_HOST` | OpenShift API URL |
| `OCP_TOKEN` | OpenShift service account token |

## Running the Bootstrap Playbook

```bash
# Set env vars for the target environment, then:
ansible-playbook playbooks/bootstrap_portal.yml -i inventories/rhdp-<customer>-<demo>/
```

## AAP Gateway OAuth — Critical Facts

The gateway's `/o/authorize/` and `/o/token/` are served by the **gateway itself** (not proxied to the controller). OAuth applications must be created in the **gateway** registry (`/api/gateway/v1/applications/`), not the controller.

**NEVER patch the client_secret after creation.** The gateway hashes secrets differently on PATCH vs POST, which always causes `invalid_client` at `/o/token/`. The only way to get valid credentials is to capture `client_secret` from the POST creation response. `bootstrap_portal.yml` deletes and recreates the app on every run to ensure fresh, valid credentials are always stored in the OCP secret.

The controller's `/api/controller/v2/applications/` and the gateway's `/api/gateway/v1/applications/` are separate registries — do not mix them.

## ansible.platform vs ansible.controller Module Map

`ansible.platform` talks to the AAP gateway API. `ansible.controller` talks to the controller API. They have different connection parameter names and different module_defaults group names.

| Task | Use | Notes |
|------|-----|-------|
| OAuth application | `ansible.platform.application` | `ansible.controller.application` is deprecated |
| Organization, team, user | `ansible.platform.organization/team/user` | Platform versions lack `galaxy_credentials` |
| Galaxy credentials on org | `ansible.controller.organization` | Only controller version supports `galaxy_credentials` |
| Credentials | `ansible.controller.credential` | No platform equivalent |
| Projects | `ansible.controller.project` / `project_update` | No platform equivalent |
| Job templates | `ansible.controller.job_template` | No platform equivalent |

**Connection parameters:**

| Collection | Host param | User param | Pass param | Cert param |
|------------|-----------|-----------|-----------|-----------|
| `ansible.platform` | `aap_hostname` | `aap_username` | `aap_password` | `aap_validate_certs` |
| `ansible.controller` | `controller_host` | `controller_username` | `controller_password` | `validate_certs` |

**module_defaults groups:**
```yaml
module_defaults:
  group/ansible.controller.controller:
    controller_host: "{{ aap_hostname }}"
    controller_username: "{{ aap_username }}"
    controller_password: "{{ aap_password }}"
    validate_certs: "{{ aap_validate_certs }}"
  group/ansible.platform.gateway:
    aap_hostname: "{{ aap_hostname }}"
    aap_username: "{{ aap_username }}"
    aap_password: "{{ aap_password }}"
    aap_validate_certs: "{{ aap_validate_certs }}"
```

Note: our inventory vars use `aap_hostname/aap_username/aap_password` — these match `ansible.platform` directly but must be mapped to `controller_*` for `ansible.controller`.

## ansible.cfg Design

There are two ansible.cfg files in play — understanding both prevents confusion:

| File | Purpose | Committed? |
|------|---------|-----------|
| `./ansible.cfg` (this repo) | Sets `collections_path`, defines galaxy server URLs — **no tokens** | Yes |
| `~/.ansible/ansible.cfg` | Personal file holding Automation Hub tokens | No — never commit |

**Why two files?** Ansible only uses one `ansible.cfg` — project-level overrides user-level entirely. So the project file can't contain tokens (they're personal and expire). Instead, server URLs live in the project file and tokens stay in the personal file.

**Precedence:** `ANSIBLE_CONFIG` env var → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. Note: `~/.ansible/ansible.cfg` (subdirectory) is NOT in Ansible's default search path — it only works because it is referenced explicitly via `ANSIBLE_CONFIG`.

**For playbook runs:** no Hub token needed — Ansible uses collections already installed at `~/.ansible/collections/`.

**For `ansible-galaxy collection install`:** the project `ansible.cfg` is active but has no tokens, so override it:
```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg ansible-galaxy collection install -r collections/requirements.yml
```

## Key Conventions

- **`ansible.platform` over `ansible.controller`** — prefer `ansible.platform` where a module exists; fall back to `ansible.controller` only when there is no platform equivalent. See the module map below.
- **Token cleanup** — stale tokens accumulate and slow AAP down over time. Always delete tokens in the same block where they are created. The only exception is the portal service token (description: `aap-selfservice-portal service token`) which is intentionally long-lived. When writing curl-based scripts, always create ONE token by capturing both `token` and `id` from a single API call — never make two separate calls. Delete it last, after all work is done:
  ```bash
  TOKEN_JSON=$(curl -s -k -X POST -u $USER:$PASS "$HOST/api/gateway/v1/tokens/" -d '{"scope":"write"}' -H "Content-Type: application/json")
  TOKEN=$(echo $TOKEN_JSON | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
  TOKEN_ID=$(echo $TOKEN_JSON | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
  # ... do work ...
  curl -s -k -X DELETE -H "Authorization: Bearer $TOKEN" "$HOST/api/gateway/v1/tokens/$TOKEN_ID/"
  ```
- **Helm chart version pinned at 2.1.0** — bump deliberately and document in CHANGELOG.md
- **Emoji status convention in docs** — use these in README and documentation files; they render natively on GitHub. Emojis are fine in docs; avoid them in code comments and Claude responses.
  - ✅ — complete, confirmed, or verified
  - ⬜ — not started or pending
  - 🔄 — in progress
- **Real inventory directories are gitignored** — `inventories/rhdp-*/` is in `.gitignore` except for `rhdp-sample-demo/` which is the template; never commit customer/environment-specific inventories
- **`docs/dev-environment.md` is gitignored** — use it for local credentials and notes; never commit it

## Architecture

The bootstrap playbook (`playbooks/bootstrap_portal.yml`) performs these steps in order:

1. Create an AAP OAuth Application via AAP Gateway API (captures `client_id`/`client_secret`)
2. Generate an AAP API token, store in OCP secret, delete token in `always:` block
3. Create OCP project `aap-portal`
4. Create OCP secrets (`secrets-rhaap-portal`, `rhaap-portal-dynamic-plugins-registry-auth`, `secrets-scm`)
5. Deploy `redhat-rhaap-portal` Helm chart from `https://charts.openshift.io`
6. Update OAuth redirect URI with the portal route
7. Poll portal `/healthz` and verify job template sync

Key Helm values to configure:
- `redhat-developer-hub.global.clusterRouterBase`
- `redhat-developer-hub.global.pluginMode: oci` — correct mode. Requires `rhaap-portal-dynamic-plugins-registry-auth` secret (see bootstrap_portal.yml). tarball mode requires `plugin-registry:8080` service not deployed by this chart.
- `redhat-developer-hub.upstream.backstage.appConfig.catalog.providers.rhaap.orgs`

## Playbook Execution Order

```
bootstrap_aap.yml    → Hub creds, project (ansible.controller)
bootstrap_portal.yml → OCP project, secrets, Helm chart, OAuth wiring (kubernetes.core)
sync_portal_orgs.yml → Query AAP for all orgs, patch portal configmap, restart pod
playbooks/site.yml   → runs all three in sequence (full from-scratch deploy)
```

## Expected Timings

End-to-end deploy via `/selfservice-bootstrap` against a fresh RHDP **Ansible Product Demo** environment. Validated 2026-05-06 against `rhdp-ames-test2`.

| Step | Duration |
|------|----------|
| `bootstrap_aap.yml` | ~50s |
| `bootstrap_portal.yml` | ~6m (Helm install + 10m wait timeout) |
| `sync_portal_orgs.yml` configmap patch | ~5s |
| Rolling pod restart after sync | ~3m |
| **Total wall clock** | **~12 min** |

Use as a sanity check when diagnosing a slow run:

- `bootstrap_aap.yml` >2m — usually a slow `Sync aap.selfservice project` because the AAP container is busy or GitHub is slow.
- `bootstrap_portal.yml` approaching 10m — the Helm `wait_timeout` is about to fire. Check `oc get pods -n aap-portal` for an init container stuck pulling from `registry.redhat.io` (auth secret issue) or a CrashLoopBackOff (config issue).
- `sync_portal_orgs.yml` configmap patch >30s — the patch isn't applying; check `oc describe configmap rhaap-portal-app-config -n aap-portal`.
- Rolling restart >5m — new pod is failing readiness; `oc logs -n aap-portal deploy/rhaap-portal -c backstage-backend --previous` is the first place to look.

All playbooks are idempotent and can be run independently.

`sync_portal_orgs.yml` must be run after `bootstrap_portal.yml` because the Helm chart sets several defaults that are wrong for a demo environment. It can also be run standalone anytime orgs, teams, or users change in AAP. It applies three fixes in one configmap patch:

| Setting | Chart default | Applied value | Why |
|---------|--------------|---------------|-----|
| `catalog.providers.rhaap.production.orgs` | `Default` only | All AAP orgs (dynamic) | Users in non-Default orgs can't log in otherwise |
| `catalog.providers.rhaap.production.sync.*.schedule.frequency` | `minutes: 60` | `minutes: 1` | 60-minute default makes portal unusable after fresh deploy |
| `permission.enabled` | `true` | `false` | RBAC defaults to deny-all for non-admins; real enforcement happens in AAP |

**Portal login requirement:** Users must be a member of an AAP organization AND a member of a team within that organization to log in to the portal. Granting a user access to a job template alone is not sufficient. `sync_portal_orgs.yml` ensures every AAP org is in the portal's allowed list; org/team membership in AAP controls who can actually authenticate.

**Template visibility:** With `permission.enabled: false`, all authenticated portal users see all synced templates. AAP enforces actual execution permissions — users can only successfully launch templates they have Execute access to in AAP.

## No Dependency on aap.as.code

This repo is fully self-contained. Do not add steps that require cloning or running
`aap.as.code` first. A user who clones only this repo must be able to reach a working
portal demo from scratch.

## Skills

All skills for this repo live in `.claude/commands/` (repo-local):

| Command | File | Purpose |
|---------|------|---------|
| `/selfservice-first-time` | `selfservice-first-time.md` | First-time prerequisite setup |
| `/selfservice-bootstrap` | `selfservice-bootstrap.md` | Generate inventory, run all three playbooks, verify |
| `/selfservice-sync` | `selfservice-sync.md` | Re-sync portal orgs/permissions/frequency after AAP changes |

Do not use `/aap-skills:aap-first-time` or `/aap-skills:aap-bootstrap` — those are
coupled to `aap.as.code` and will fail in this repo.

See `ROADMAP.md` for the full phased implementation plan.
