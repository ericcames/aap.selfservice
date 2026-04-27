# aap.selfservice — Roadmap

Repeatable build plan for the AAP self-service automation portal on RHDP-provisioned OpenShift.

## Goal

After provisioning an **Ansible Product Demo** on the Red Hat Demo Platform (RHDP), run one
playbook to install and configure the self-service automation portal so it is demo-ready.
The RHDP environment provides:
- AAP 2.6 (Controller, Hub, EDA, Gateway) — admin credentials provided
- OpenShift 4.x — API endpoint and service account token provided

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

### Phase 1 — Bootstrap Playbook

Build `playbooks/bootstrap_portal.yml` that automates the full install guide:
[https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal/index](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal/index)

**Tasks:**

1. **Create AAP OAuth Application**
   - POST to AAP Controller API
   - Capture `client_id` and `client_secret`
   - Set redirect URI to portal URL (update after deploy)

2. **Generate AAP API Token**
   - POST to AAP Controller API
   - Store in OCP secret
   - Delete token in `always:` block (or persist in secret and skip deletion)

3. **Create OCP Project**
   - `oc new-project aap-portal` (idempotent)

4. **Create OCP Secrets**
   - `aap-auth` — AAP URL, client_id, client_secret, API token
   - `git-auth` — GitHub/GitLab PAT for template content sources

5. **Deploy Helm Chart**
   - Chart: `redhat-rhaap-portal` from `https://charts.openshift.io`
   - Version: 2.1.0 (pin, update deliberately)
   - Key values:
     - `redhat-developer-hub.global.clusterRouterBase`
     - `redhat-developer-hub.global.pluginMode: oci`
     - `redhat-developer-hub.upstream.backstage.appConfig.catalog.providers.rhaap.orgs`

6. **Update OAuth Redirect URI**
   - Once portal route is known, PATCH the AAP OAuth application with the portal URL

7. **Verify Deployment**
   - Poll portal `/healthz` or check pod readiness
   - Confirm job template sync from Controller

**Collections needed:**
- `ansible.platform` — AAP API interactions
- `kubernetes.core` — OCP secrets and Helm chart deployment

### Phase 2 — RBAC Configuration

Automate the initial RBAC setup from the configuration guide:
[https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/configuring_self-service_automation_portal/index](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/configuring_self-service_automation_portal/index)

- Define which AAP teams/organizations sync to portal
- Set template display logic (which templates are visible to which users)
- Configure conditional access rules

### Phase 3 — Demo Content

- Create a sample Git repository of Backstage software templates
- Templates map to the standard demo job templates (Linux, Windows, F5, etc.)
- Templates use guided forms so non-Ansible users can self-service

### Phase 4 — Integration with aap.as.code

- Add a `Setup - Self-Service Portal` job template to `aap.as.code` CaC
- Template calls this repo's playbook from within AAP
- Makes portal setup part of the standard demo provisioning flow

## Inventory Design

Each RHDP environment gets its own named inventory (same pattern as `aap.as.code`):

```
inventories/
└── rhdp-<customer>-<demo>/
    └── group_vars/
        └── all.yml     # reads CONTROLLER_HOST, OCP_HOST, OCP_TOKEN from env
```

Sensitive values are never stored in the inventory — resolved at runtime via env vars.

## Open Questions

- Which AAP organization should be synced by default for demos? (`Default` org assumed)
- Should Git PAT be required at bootstrap, or can template content be deferred to Phase 3?
- Should the portal token be long-lived (stored in OCP secret) or short-lived (generated per run)?
