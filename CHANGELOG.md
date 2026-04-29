# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- `playbooks/bootstrap_aap.yml` — configures AAP for portal: Hub credentials, vault credential, aap.selfservice project, OAuth application

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
- `.gitignore` updated to track `collections/requirements.yml`

## [0.1.0] — 2026-04-27

### Added
- Initial repository structure
- README with architecture overview, Helm chart details, and documentation links
- ROADMAP with phased implementation plan based on live cluster discovery against RHDP Ansible Product Demo
