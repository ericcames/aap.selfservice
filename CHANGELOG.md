# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- `playbooks/sync_portal_orgs.yml` — queries AAP for all current organizations and patches the portal configmap live; run after `bootstrap_portal.yml` or standalone anytime orgs/teams/users change in AAP
- `site.yml` — full from-scratch deploy: runs `bootstrap_aap.yml`, `bootstrap_portal.yml`, and `sync_portal_orgs.yml` in sequence
- `.claude/commands/selfservice-sync.md` — new skill for re-syncing portal org list, permissions, and sync frequency after AAP changes
- `playbooks/bootstrap_aap.yml` — configures AAP for portal: Hub credentials, vault credential, aap.selfservice project, OAuth application
- `playbooks/bootstrap_portal.yml` — deploys redhat-rhaap-portal Helm chart on OpenShift with full OCI plugin auth and OAuth wiring
- `ansible.cfg` — project-level galaxy server config (no tokens; tokens stay in `~/.ansible/ansible.cfg`)
- `collections/requirements.yml` — pinned collection versions (ansible.platform, ansible.controller, kubernetes.core)
- `CLAUDE.md` — project guidance for Claude Code including ansible.cfg design explanation
- `.claude/commands/selfservice-first-time.md` — local skill for first-time prerequisite setup
- `.claude/commands/selfservice-bootstrap.md` — local skill for inventory generation, playbook execution, and verification
- `.claude/commands/references/context.md` — shared reference for AAP object names, API endpoints, and env vars
- `inventories/rhdp-sample-demo/` — template inventory for new RHDP environments
- `docs/images/redhatdemo.png` — Ansible Product Demo catalog item screenshot for onboarding

### Removed
- AAP vault credential and `~/.ansible/secrets2` are no longer required to run this repo. Nothing in this repo's playbooks uses ansible-vault encrypted content, and no job templates reference the credential. Removed the `Create vault credential` task from `bootstrap_aap.yml`; dropped `secrets2` setup (Step 3) and `User Identity Defaults` (Step 7) from `selfservice-first-time`; dropped `--vault-password-file ~/.ansible/secrets2` from all `ansible-playbook` invocations in `selfservice-bootstrap` and `selfservice-sync`; dropped `vault_password`, `my_vault`, and the vault verify block from skill output and the sample inventory; updated README and CLAUDE.md (resolves #32)
- `selfservice-first-time` skill: dropped Step 7 (SSH key setup) and Step 8 (hosted vault URL) along with the `my_remote_vault`, `my_remote_ssh_pub_key`, and `my_windows_catalog_short_description` prompts — none of these are referenced by any playbook in this repo (resolves #29)
- `selfservice-bootstrap` skill: dropped the same vars from the read-defaults block and the inventory generation template; also dropped the unused `aap_organization` for consistency with the sample inventory (#28)

### Fixed
- `selfservice-first-time` skill: replaced stale `/aap-bootstrap` references with `/selfservice-bootstrap` in the orientation header and final-validation success message (resolves #29)
- `inventories/rhdp-sample-demo/group_vars/all.yml`: variable names now match what playbooks read (`aap_*` instead of `controller_*`); added required vars (`aap_validate_certs`, `hub_token`, `hub_auth_url`, `vault_password`, `my_vault`); removed unused `aap_organization`. Anyone copying this template can now run the playbooks without inventory edits beyond setting env vars (resolves #28)
- Security: added `no_log: true` to tasks in `bootstrap_portal.yml` that register OAuth client_secret, AAP bearer token, and cluster pull secret — previously exposed in plaintext when running with `-v` (resolves #26)
- Portal login failure for users in non-Default AAP organizations: Helm chart defaults `catalog.providers.rhaap.production.orgs` to `Default` only; `sync_portal_orgs.yml` now patches the configmap with the full org list queried live from AAP (resolves #18)
- Templates not visible to non-admin users: Helm chart enables Backstage RBAC (`permission.enabled: true`) with no policies, which defaults to deny-all for non-admins; `sync_portal_orgs.yml` disables permission enforcement — actual access control is enforced by AAP at job launch time (resolves #18)
- Catalog not populated for 60 minutes after deploy: Helm chart defaults both sync providers to `frequency: minutes: 60`; `sync_portal_orgs.yml` patches both to `minutes: 1` so templates and users appear within one minute of deployment (resolves #18)
- `selfservice-bootstrap` skill updated to include `sync_portal_orgs.yml` as a required step after portal deploy (resolves #20)
- `selfservice-bootstrap` skill: corrected stale skill name references (`/aap-first-time` → `/selfservice-first-time`, `/aap-bootstrap` → `/selfservice-bootstrap`)
- README: removed inaccurate claim that RHDH is included with an AAP subscription (RHDH requires a separate entitlement)

### Changed
- ROADMAP updated: Phase 1 complete, playbook execution order reflects all three playbooks, open questions resolved, `site.yml` exists
- README updated: bastion removed from prerequisites (optional, not required), repository structure reflects all current files, skills list updated
- ROADMAP updated with architecture decisions: self-contained repo (no aap.as.code dependency), local skills, phased build order
- README updated with collection install instructions and RHDP catalog item screenshot

## [0.1.0] — 2026-04-27

### Added
- Initial repository structure
- README with architecture overview, Helm chart details, and documentation links
- ROADMAP with phased implementation plan based on live cluster discovery against RHDP Ansible Product Demo
