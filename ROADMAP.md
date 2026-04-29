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

### Phase 0 — Repository Setup (Complete)

- [x] Create `aap.selfservice` repo
- [x] Document architecture and plan in README and ROADMAP
- [x] Capture documentation links
- [x] Add `ansible.cfg` (no tokens), `collections/requirements.yml`, CLAUDE.md
- [x] Establish inventory pattern (`inventories/rhdp-<customer>-<demo>/`)

### Phase 1 — Local Skills

Build Claude Code skills in `.claude/commands/` so a user arriving at this repo for the
first time has everything they need without relying on published plugins.

- [x] `/selfservice-first-time` — verify and walk through all local prerequisites
- [ ] `/selfservice-bootstrap` — generate inventory, run playbooks, verify results

### Phase 2 — AAP Bootstrap Playbook (Complete)

`playbooks/bootstrap_aap.yml` — minimum AAP configuration needed for the portal
to function. This replaces the dependency on `aap.as.code`.

- [x] Hub credentials (`Automation Hub - certified`, `Automation Hub - validated`) assigned to Default Organization
- [x] Vault credential
- [x] `aap.selfservice` project (synced from `main`)
- [x] AAP OAuth Application for portal authentication

**Collections:** `ansible.platform` (OAuth app), `ansible.controller` (credentials, project)

### Phase 3 — Portal Bootstrap Playbook

Build `playbooks/bootstrap_portal.yml` — full portal install on OpenShift.
Reference: [Installing Self-Service Automation Portal](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal/index)

**Tasks:**
1. Create OCP project (`aap-portal`)
2. Create OCP secrets (`aap-auth`, `git-auth`)
3. Deploy `redhat-rhaap-portal` Helm chart (v2.1.0, pinned) from `https://charts.openshift.io`
   - Key values: `clusterRouterBase`, `pluginMode: oci`, `catalog.providers.rhaap.orgs`
4. Update AAP OAuth redirect URI with portal route
5. Poll portal `/healthz` and verify job template sync

**Collections:** `kubernetes.core`

### Phase 4 — Demo Content

Define and automate demo personas and content so the portal tells a compelling customer story.
Exact job templates and user personas TBD — decided when portal integration is validated.

Goals:
- Multiple user personas with different permission levels (e.g. operator, DBA, developer)
- Each persona sees only the job templates relevant to their role
- Templates use guided forms so non-Ansible users can self-service

### Phase 5 — RBAC Configuration

Automate the RBAC setup from the configuration guide:
[Configuring Self-Service Automation Portal](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/configuring_self-service_automation_portal/index)

- Define which AAP teams/organizations sync to portal
- Set template display logic per persona
- Configure conditional access rules

## Playbook Execution Order

```
bootstrap_aap.yml      → configure AAP (Hub creds, vault, project, OAuth app)
bootstrap_portal.yml   → install portal on OCP and wire to AAP
```

Both playbooks are idempotent and can be run independently. A top-level `site.yml`
will orchestrate them in sequence for a full from-scratch deploy.

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

- Which AAP organization should be synced by default for demos? (`Default` org assumed)
- Should Git PAT be required at bootstrap, or can template content be deferred to Phase 4?
- Should the portal token be long-lived (stored in OCP secret) or short-lived (generated per run)?
