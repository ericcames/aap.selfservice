# aap.selfservice

Repeatable build process for the **Red Hat Ansible Automation Platform Self-Service Automation Portal** on OpenShift Container Platform.

## What Is the Self-Service Automation Portal?

The self-service automation portal gives end users a point-and-click web interface to run automation without needing to understand Ansible playbooks or have direct AAP access. It is built on **Red Hat Developer Hub (RHDH)** and deployed as a Helm chart on OpenShift.

![Portal Templates View](docs/images/portal-logged-in.png)

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
3. **Bastion host** — SSH access provided by RHDP Ansible Product Demo
4. **`oc` CLI** — logged in to the OCP cluster
5. **`helm` CLI** — available locally
6. **`~/.ansible/secrets2`** — vault password file (one line: your vault password)
7. **Ansible collections** — install once, shared across all repos. Your Automation Hub token must be in `~/.ansible/ansible.cfg` under `[galaxy_server.rh_certified]`. Get it from [console.redhat.com](https://console.redhat.com) → Automation Hub → Connect to Hub → API token. Then install:

```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg ansible-galaxy collection install -r collections/requirements.yml
```

Run `/selfservice-first-time` for a guided setup if this is a new machine.

## Getting Started

This repo is self-contained. Clone it, run `/selfservice-first-time` in Claude Code to verify prerequisites, then run `/selfservice-bootstrap` to deploy.

```
Step 1 → /selfservice-first-time   (verify local prerequisites)
Step 2 → /selfservice-bootstrap    (configure AAP + install portal)
```

## Repository Structure

```
aap.selfservice/
├── playbooks/
│   ├── bootstrap_aap.yml      # configures AAP (Hub creds, vault, project)
│   └── bootstrap_portal.yml   # deploys portal on OpenShift
├── docs/
│   ├── images/                # screenshots and diagrams
│   └── dev-environment.md     # local credentials — gitignored, never commit
├── collections/
│   └── requirements.yml       # pinned collection versions
├── inventories/
│   └── rhdp-sample-demo/      # template inventory — copy for each environment
├── ansible.cfg                # galaxy server config (no tokens)
├── README.md
├── ROADMAP.md
└── CHANGELOG.md
```
