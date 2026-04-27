# aap.selfservice

Repeatable build process for the **Red Hat Ansible Automation Platform Self-Service Automation Portal** on OpenShift Container Platform.

## What Is the Self-Service Automation Portal?

The self-service automation portal gives end users a point-and-click web interface to run automation without needing to understand Ansible playbooks or have direct AAP access. It is built on **Red Hat Developer Hub (RHDH)** and deployed as a Helm chart on OpenShift.

Key facts:
- Built on RHDH — included in your AAP subscription, no separate RHDH license required
- Syncs Organizations, Users, Teams, and Job Templates from AAP Controller
- Git-backed software templates define what end users can run
- RBAC controls which users can see and launch which templates
- Deployed via the `redhat-rhaap-portal` Helm chart from `charts.openshift.io`

## Documentation

| Guide | URL |
|-------|-----|
| Installing Self-Service Automation Portal | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal/index |
| Configuring Self-Service Automation Portal | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/configuring_self-service_automation_portal/index |

## Architecture

```
RHDP Ansible Product Demo
├── AAP 2.6 (already provisioned by RHDP)
│   ├── Automation Controller  ← source of job templates
│   ├── Automation Hub
│   └── Event-Driven Ansible
└── OpenShift (already provisioned by RHDP)
    └── aap.selfservice (this repo)
        └── redhat-rhaap-portal Helm chart
            └── RHDH instance + AAP plugin
```

## Helm Chart

| Field | Value |
|-------|-------|
| Chart name | `redhat-rhaap-portal` |
| Display name | AAP self-service automation portal |
| Repository | `https://charts.openshift.io` |
| Latest version | 2.1.0 |
| Provider | Red Hat |

## Prerequisites

Before running the bootstrap playbook you need:

1. **AAP Controller** — running and accessible (provided by RHDP Ansible Product Demo)
2. **OpenShift** — API access with cluster-admin (provided by RHDP Ansible Product Demo)
3. **AAP OAuth Application** — created in AAP Controller (playbook handles this)
4. **AAP API Token** — for portal-to-controller authentication (playbook handles this)
5. **Git Personal Access Token** — GitHub or GitLab PAT for content/template sources
6. **`oc` CLI** — logged in to the OCP cluster
7. **`helm` CLI** — available locally
8. **`~/.ansible/secrets2`** — vault password file (same as `aap.as.code`)

## Setup Steps (High Level)

See [ROADMAP.md](ROADMAP.md) for the full implementation plan.

1. Create AAP OAuth application and API token
2. Create OCP project (`aap-portal`)
3. Create OCP secrets for AAP credentials and Git PAT
4. Deploy `redhat-rhaap-portal` Helm chart
5. Add portal URL to AAP OAuth application redirect URIs
6. Configure RBAC for end users
7. Verify sync of job templates from AAP Controller

## Repository Structure

```
aap.selfservice/
├── playbooks/
│   └── bootstrap_portal.yml   # end-to-end install playbook (planned)
├── docs/
│   └── dev-environment.md     # local credentials — gitignored, never commit
├── README.md
├── ROADMAP.md
└── CHANGELOG.md
```

## Relationship to aap.as.code

This repo is a follow-on to [aap.as.code](https://github.com/ericcames/aap.as.code).
Run `aap.as.code` bootstrap and CaC setup first, then run this repo's bootstrap to
layer the self-service portal on top.

```
Step 1 → aap.as.code/playbooks/bootstrap_dev.yml      (sets up AAP)
Step 2 → aap.as.code Setup - AAP - CAC job template   (loads demo config)
Step 3 → aap.selfservice/playbooks/bootstrap_portal.yml (installs portal)
```
