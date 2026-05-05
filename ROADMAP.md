# aap.selfservice — Roadmap

Repeatable build plan for the AAP self-service automation portal on RHDP-provisioned OpenShift.

## Goal

After provisioning an **Ansible Product Demo** on the Red Hat Demo Platform (RHDP), run one
playbook to install and configure the self-service automation portal so it is demo-ready.
The RHDP environment provides:
- AAP 2.6 (Controller, Hub, EDA, Gateway) — admin credentials provided
- OpenShift 4.x — API endpoint and service account token provided

**This repo stands alone.** There is no dependency on `aap.as.code` or any other repo.
A user who clones only this repo can bootstrap a complete, demo-ready portal.

## Architecture Decisions (2026-04-29)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Repo dependency | None — fully self-contained | Demo repos that require other repos fail at customer sites |
| AAP bootstrap | This repo configures AAP from scratch | Can't assume aap.as.code was run first |
| Skills | All local in `.claude/commands/` | Skills evolve with the playbooks; repo-local keeps them in sync |
| Collections | Installed at `~/.ansible/collections/` | Shared across all repos; no per-project reinstall |
| Demo content | Deferred — decided in Phase 3 | Portal integration is independent of what templates exist |

## Cluster Facts (Discovered 2026-04-27)

The following was confirmed against a live RHDP Ansible Product Demo cluster:

| Component | Status | Notes |
|-----------|--------|-------|
| AAP Operator | Installed (v2.6.0) | namespace: `aap` |
| Automation Controller | Running | `aap-controller-aap.apps.<cluster>` |
| Automation Hub | Running | `aap-hub-aap.apps.<cluster>` |
| Event-Driven Ansible | Running | `aap-eda-aap.apps.<cluster>` |
| AAP Gateway | Running | `aap-aap.apps.<cluster>` |
| `redhat-rhaap-portal` Helm chart | Available (v2.1.0) | in `charts.openshift.io` — **not installed** |
| RHDH Operator | Available (v1.9.3) | not needed — Helm chart is self-contained |

## Implementation Phases

### Phase 0 — Repository Setup ✅

- ✅ Create `aap.selfservice` repo
- ✅ Document architecture and plan in README and ROADMAP
- ✅ Capture documentation links
- ✅ Add `ansible.cfg` (no tokens), `collections/requirements.yml`, CLAUDE.md
- ✅ Establish inventory pattern (`inventories/rhdp-<customer>-<demo>/`)

### Phase 1 — Local Skills ✅

Build Claude Code skills in `.claude/commands/` so a user arriving at this repo for the
first time has everything they need without relying on published plugins.

- ✅ `/selfservice-first-time` — verify and walk through all local prerequisites
- ✅ `/selfservice-bootstrap` — generate inventory, run all three playbooks, verify results
- ✅ `/selfservice-sync` — re-sync portal org list, permissions, and sync frequency after AAP changes

### Phase 2 — AAP Bootstrap Playbook ✅

`playbooks/bootstrap_aap.yml` — minimum AAP configuration needed for the portal
to function. This replaces the dependency on `aap.as.code`.

- ✅ Hub credentials (`Automation Hub - certified`, `Automation Hub - validated`) assigned to Default Organization
- ✅ Vault credential
- ✅ `aap.selfservice` project (synced from `main`)
- ✅ AAP OAuth Application for portal authentication

**Collections:** `ansible.platform` (OAuth app via gateway API), `ansible.controller` (credentials, project — no platform equivalent exists)

### Phase 3 — Portal Bootstrap Playbook ✅

`playbooks/bootstrap_portal.yml` — full portal install on OpenShift. Tested and working.

- ✅ clusterRouterBase derived from OCP API URL
- ✅ OAuth app client_secret rotated and stored in OCP secret
- ✅ AAP service token created (block/rescue cleanup on failure)
- ✅ OCP namespace, secrets-rhaap-portal, secrets-scm
- ✅ `rhaap-portal-dynamic-plugins-registry-auth` secret (cluster pull secret → init container auth)
- ✅ `redhat-rhaap-portal` Helm chart deployed (v2.1.0, pluginMode: oci)
- ✅ OAuth redirect URI updated with real portal route
- ✅ Portal running at: `https://rhaap-portal-aap-portal.apps.<cluster>`

**Verified working:** Login → Templates view → Demo Job Template synced from AAP ✅

**Key findings documented in CLAUDE.md:**
- `pluginMode: oci` requires `rhaap-portal-dynamic-plugins-registry-auth` secret (cluster pull secret)
- GitHub/GitLab integrations must be explicitly disabled (`integrations.github: []`) when no SCM token provided
- Gateway app client_secret must be captured at POST creation — never PATCH it (different hashing causes `invalid_client`)
- Gateway `/o/authorize/` and `/o/token/` use the gateway app registry, NOT the controller's

Build `playbooks/bootstrap_portal.yml` — full portal install on OpenShift.
Reference: [Installing Self-Service Automation Portal](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal/index)

**Tasks:**
1. Create OCP project (`aap-portal`)
2. Create OCP secrets (`aap-auth`, `git-auth`)
3. Deploy `redhat-rhaap-portal` Helm chart (v2.1.0, pinned) from `https://charts.openshift.io`
   - Key values: `clusterRouterBase`, `pluginMode: oci`
   - Note: `catalog.providers.rhaap.orgs` cannot be cleanly overridden via Helm values (chart uses a template expression as a YAML key causing DUPLICATE_KEY on merge); patched post-deploy by `sync_portal_orgs.yml` instead
4. Update AAP OAuth redirect URI with portal route
5. Poll portal `/healthz` and verify job template sync

**Collections:** `kubernetes.core`

### Phase 4 — Demo Content 🔄

> Planning in progress — see [Issue #1](https://github.com/ericcames/aap.selfservice/issues/1) for the planning discussion. Implementation will not start until that issue is resolved.

Define and automate demo personas and content so the portal tells a compelling customer story.
Exact job templates and user personas TBD — decided when portal integration is validated.

Goals:
- Multiple user personas with different permission levels (e.g. operator, DBA, developer)
- Each persona sees only the job templates relevant to their role
- Templates use guided forms so non-Ansible users can self-service

### Phase 5 — RBAC Configuration ⬜

Automate the RBAC setup from the configuration guide:
[Configuring Self-Service Automation Portal](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/configuring_self-service_automation_portal/index)

- Define which AAP teams/organizations sync to portal
- Set template display logic per persona
- Configure conditional access rules

## Playbook Execution Order

```
bootstrap_aap.yml      → configure AAP (Hub creds, vault, project, OAuth app)
bootstrap_portal.yml   → install portal on OCP and wire to AAP
sync_portal_orgs.yml   → patch portal configmap: org list, permissions, sync frequency
```

All three playbooks are idempotent and can be run independently. `site.yml` orchestrates
all three in sequence for a full from-scratch deploy. Run `/selfservice-bootstrap` to
execute the full sequence with guided credential collection and verification.

## Inventory Design

Each RHDP environment gets its own named inventory:

```
inventories/
└── rhdp-<customer>-<demo>/
    ├── hosts
    └── group_vars/
        └── all.yml     # reads CONTROLLER_HOST, OCP_HOST, OCP_TOKEN from env
```

Sensitive values are never stored in the inventory — resolved at runtime via env vars.
See `inventories/rhdp-sample-demo/` for the template.

## Open Questions

- Should Git PAT be required at bootstrap, or can template content be deferred to Phase 4?

## Resolved Questions

- **Which AAP organizations should the portal sync?** — `sync_portal_orgs.yml` queries AAP at run time and syncs all current organizations dynamically. No hardcoded list.
- **Should the portal token be long-lived or short-lived?** — Long-lived (stored in OCP secret as `aap-selfservice-portal service token`). Short-lived tokens would expire and break the portal backend.
- **How does portal login work for users not in the Default org?** — Users must be a member of an AAP organization AND a team within that org. `sync_portal_orgs.yml` ensures every org is in the allowed list. Template access is visible to all authenticated users; AAP enforces execute permissions at job launch time.
