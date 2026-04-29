# aap.selfservice Context Reference

Shared reference for local skills in this repo.

---

## AAP Object Names

Objects created by `bootstrap_aap.yml`:

| Object | Type | Name |
|--------|------|------|
| Hub certified credential | Ansible Galaxy/Automation Hub API Token | `Automation Hub - certified` |
| Hub validated credential | Ansible Galaxy/Automation Hub API Token | `Automation Hub - validated` |
| Vault credential | Vault | varies by user — query by credential type |
| Project | Git | `aap.selfservice` |
| OAuth Application | OAuth2 Provider Application | `aap-selfservice-portal` |

---

## AAP API Endpoints

AAP 2.5+ uses a gateway architecture:

| Purpose | Endpoint |
|---------|----------|
| Token create/delete | `$CONTROLLER_HOST/api/gateway/v1/tokens/` |
| Credentials | `$CONTROLLER_HOST/api/controller/v2/credentials/` |
| Credential types | `$CONTROLLER_HOST/api/controller/v2/credential_types/` |
| Projects | `$CONTROLLER_HOST/api/controller/v2/projects/` |
| Job templates | `$CONTROLLER_HOST/api/controller/v2/job_templates/` |
| OAuth applications | `$CONTROLLER_HOST/api/controller/v2/applications/` |

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `CONTROLLER_HOST` | AAP Gateway URL (e.g. `https://aap-aap.apps.<cluster>`) |
| `CONTROLLER_USERNAME` | AAP admin username (default: `admin`) |
| `CONTROLLER_PASSWORD` | AAP admin password |
| `OCP_HOST` | OpenShift API URL |
| `OCP_TOKEN` | OpenShift service account token |

---

## Inventory Pattern

```
inventories/rhdp-<customer>-<demo>/
├── hosts
└── group_vars/
    └── all.yml    # no secrets — all sensitive values resolved via lookups at runtime
```

Copy `inventories/rhdp-sample-demo/` as a starting point.

---

## Hosted File URL Patterns

Vault files and SSH public keys must be at public HTTPS raw URLs:

```
✅ https://raw.githubusercontent.com/user/repo/main/vault.yml
❌ https://github.com/user/repo/blob/main/vault.yml   (returns HTML)
❌ https://github.com/user/repo/raw/main/vault.yml    (redirects)
```
