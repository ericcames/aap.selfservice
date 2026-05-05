# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- `playbooks/sync_portal_orgs.yml` — queries AAP for all current organizations and patches the portal configmap live; run after `bootstrap_portal.yml` or standalone anytime orgs/teams/users change in AAP
- `site.yml` — full from-scratch deploy: runs `bootstrap_aap.yml`, `bootstrap_portal.yml`, and `sync_portal_orgs.yml` in sequence

### Fixed
- Portal login failure for users in non-Default AAP organizations: Helm chart defaults `catalog.providers.rhaap.production.orgs` to `Default` only; `sync_portal_orgs.yml` now patches the configmap with the full org list queried live from AAP (resolves issue #18)
- Templates not visible to non-admin users: Helm chart enables Backstage RBAC (`permission.enabled: true`) with no policies, which defaults to deny-all for non-admins; `sync_portal_orgs.yml` disables permission enforcement — actual access control is enforced by AAP at job launch time (resolves issue #18)
- Catalog not populated for 60 minutes after deploy: Helm chart defaults both sync providers to `frequency: minutes: 60`; `sync_portal_orgs.yml` patches both to `minutes: 1` so templates and users appear within one minute of deployment (resolves issue #18)
- `selfservice-bootstrap` skill: corrected stale skill name references (`/aap-first-time` → `/selfservice-first-time`, `/aap-bootstrap` → `/selfservice-bootstrap`)
- README: removed inaccurate claim that RHDH is included with an AAP subscription (RHDH requires a separate entitlement)

### Changed
- ROADMAP Phase 1 marked complete

### Added
- `playbooks/bootstrap_aap.yml` — configures AAP for portal: Hub credentials, vault credential, aap.selfservice project, OAuth application
- `playbooks/bootstrap_portal.yml` — deploys redhat-rhaap-portal Helm chart on OpenShift with full OCI plugin auth and OAuth wiring

### Added
- `ansible.cfg` — project-level galaxy server config (no tokens; tokens stay in `~/.ansible/ansible.cfg`)
- `collections/requirements.yml` — pinned collection versions (ansible.platform, ansible.controller, kubernetes.core)
- `CLAUDE.md` — project guidance for Claude Code including ansible.cfg design explanation
- `.claude/commands/selfservice-first-time.md` — local skill for first-time prerequisite setup
- `.claude/commands/selfservice-bootstrap.md` — local skill for inventory generation, playbook execution, and verification
- `.claude/commands/references/context.md` — shared reference for AAP object names, API endpoints, and env vars
- `inventories/rhdp-sample-demo/` — template inventory for new RHDP environments

### Changed
- ROADMAP updated with architecture decisions: self-contained repo (no aap.as.code dependency), local skills, phased build order
- README updated with collection install instructions
- README Prerequisites section now includes RHDP catalog item screenshot and link so new users know which demo to order
- `.gitignore` updated to track `collections/requirements.yml`

### Added
- `docs/images/redhatdemo.png` — Ansible Product Demo catalog item screenshot for onboarding

## [0.1.0] — 2026-04-27

### Added
- Initial repository structure
- README with architecture overview, Helm chart details, and documentation links
- ROADMAP with phased implementation plan based on live cluster discovery against RHDP Ansible Product Demo
