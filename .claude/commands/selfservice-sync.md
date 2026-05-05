# AAP Self-Service Sync

Run this skill after any org, team, or user changes in AAP to sync the portal's allowed org list and restart the portal. Also safe to run anytime the portal is showing stale data or after a `bootstrap_portal.yml` re-run.

What it does in one configmap patch:
- Updates `catalog.providers.rhaap.production.orgs` with all current AAP organizations (queried live)
- Keeps `permission.enabled: false` so all authenticated users can see templates
- Keeps sync frequency at `minutes: 1` so changes appear quickly

## Preflight Check

```bash
# AAP reachable
echo "CONTROLLER_HOST=${CONTROLLER_HOST}"
echo "CONTROLLER_USERNAME=${CONTROLLER_USERNAME}"
echo "CONTROLLER_PASSWORD=${CONTROLLER_PASSWORD}"

# OCP logged in
oc whoami 2>/dev/null && echo "✅ oc — logged in" || echo "❌ oc — not logged in"

# portal namespace exists
oc get namespace aap-portal 2>/dev/null && echo "✅ aap-portal namespace" || echo "❌ aap-portal namespace — run /selfservice-bootstrap first"

# portal configmap exists
oc get configmap rhaap-portal-app-config -n aap-portal 2>/dev/null && echo "✅ configmap found" || echo "❌ configmap not found — run /selfservice-bootstrap first"
```

If env vars are missing, ask the user which inventory to use and remind them to export:
```bash
export CONTROLLER_HOST=<aap_url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>
export OCP_HOST=<ocp_api_url>
export OCP_TOKEN=<token>
oc login $OCP_HOST --token=$OCP_TOKEN --insecure-skip-tls-verify=true
```

## Step 1 — Identify the Inventory

Ask which inventory to use (lists available options):
```bash
ls inventories/ | grep rhdp
```

## Step 2 — Run sync_portal_orgs.yml

Run immediately — no confirmation needed:

```bash
ansible-playbook -i inventories/rhdp-<customer>-<demo>/ playbooks/sync_portal_orgs.yml \
  --vault-password-file ~/.ansible/secrets2
```

## Step 3 — Confirm

Show the org list from the playbook output and confirm the portal restarted cleanly:

```bash
oc get pods -n aap-portal
```

Expected: one `rhaap-portal-*` pod `Running 2/2` and one `rhaap-portal-postgresql-*` pod `Running 1/1`.

Report back:
```
Sync complete.

Portal org list updated: [list orgs from playbook output]

Portal: https://rhaap-portal-aap-portal.apps.<cluster>

Templates and users will appear within ~1 minute as the catalog syncs.
```

### On failure

If the playbook fails with `rhaap-portal-app-config not found`, the portal hasn't been deployed yet. Run `/selfservice-bootstrap` first.

If the OCP token has expired, ask the user to get a fresh token from the RHDP Services page and re-export `OCP_TOKEN`, then re-run.
