# AAP Self-Service First-Time Setup

This skill walks a new user through every local prerequisite needed to run `/selfservice-bootstrap` in this repo. Check what already exists, guide creation of anything missing, and end with a full preflight validation.

Run this once on a new machine. If you've already done this setup, skip straight to `/selfservice-bootstrap`.

## Orientation

Print this once at the start:

```
You're setting up prerequisites to deploy the AAP Self-Service Portal from your laptop.
This takes about 10 minutes the first time.

We'll set up:
  1. Your Automation Hub API token (~/.ansible/ansible.cfg)
  2. Required Ansible collections (ansible.platform, ansible.controller, kubernetes.core)
  3. oc CLI — logged in to OpenShift
  4. helm CLI — available locally

You'll also need your RHDP environment credentials handy:
  - AAP URL + password (from RHDP Services page)
  - OCP API URL + token (from RHDP Services page)

  Note: the bastion host is NOT required. Bootstrap runs from your laptop directly.
  The bastion is available for troubleshooting if you can't reach the OCP API.

After this, you'll be ready to run /selfservice-bootstrap.
```

Verify the user is in the `aap.selfservice` repo:

```bash
test -f collections/requirements.yml && echo "✅ In aap.selfservice directory" || echo "❌ Wrong directory"
```

If the check fails, stop and tell the user:

```
❌ You must run this skill from inside the aap.selfservice repo.

Clone it first:
  git clone https://github.com/ericcames/aap.selfservice.git
  cd aap.selfservice
  claude .

Then re-run /selfservice-first-time.
```

## Step 1 — Check What Already Exists

Audit current state before doing anything:

```bash
# ansible.cfg
grep -q "galaxy_server.rh_certified" ~/.ansible/ansible.cfg 2>/dev/null && \
  grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "EXISTS: ansible.cfg with Hub token" || echo "MISSING: ansible.cfg or Hub token"

# collections
test -d ~/.ansible/collections/ansible_collections/ansible/platform && \
  echo "EXISTS: ansible.platform" || echo "MISSING: ansible.platform"
test -d ~/.ansible/collections/ansible_collections/ansible/controller && \
  echo "EXISTS: ansible.controller" || echo "MISSING: ansible.controller"
ansible-galaxy collection list 2>/dev/null | grep -q "^kubernetes.core" && \
  echo "EXISTS: kubernetes.core" || echo "MISSING: kubernetes.core"

# oc CLI
oc whoami 2>/dev/null && echo "EXISTS: oc — logged in as $(oc whoami)" || echo "MISSING: oc not found or not logged in"

# helm CLI
helm version --short 2>/dev/null && echo "EXISTS: helm" || echo "MISSING: helm"
```

Print a summary of what was found before proceeding. Skip any step below where the item already exists and is valid.

## Step 2 — ansible.cfg Setup

Skip if `~/.ansible/ansible.cfg` exists with a valid Hub token.

If missing or token is a placeholder, ask:
> "Paste your Automation Hub API token. Get it at: https://console.redhat.com/ansible/automation-hub/token"

Write `~/.ansible/ansible.cfg` with this template, substituting the token into both server sections:

```ini
[defaults]

[galaxy]
server_list = rh_certified, rh_validated, community

[galaxy_server.rh_certified]
url=https://console.redhat.com/api/automation-hub/content/published/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=<USER_TOKEN>

[galaxy_server.rh_validated]
url=https://console.redhat.com/api/automation-hub/content/validated/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=<USER_TOKEN>

[galaxy_server.community]
url=https://galaxy.ansible.com/
```

```bash
chmod 600 ~/.ansible/ansible.cfg
grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "✅ ~/.ansible/ansible.cfg written" || echo "❌ token not set correctly"
```

## Step 3 — Collections Install

Skip if all three collections are found by `ansible-galaxy collection list` (they may be at `~/.ansible/collections/` or the system Python path).

The Hub token must be in `~/.ansible/ansible.cfg` — never pass it on the command line. Show this command and ask for confirmation before running:

```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
  ansible-galaxy collection install -r collections/requirements.yml
```

This installs to `~/.ansible/collections/` — shared across all your repos. After running, validate:

```bash
test -d ~/.ansible/collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform" || echo "❌ ansible.platform — MISSING"
test -d ~/.ansible/collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller" || echo "❌ ansible.controller — MISSING"
ansible-galaxy collection list 2>/dev/null | grep -q "^kubernetes.core" && \
  echo "✅ kubernetes.core" || echo "❌ kubernetes.core — MISSING"
```

If any are missing after install, report the error and stop.

## Step 4 — oc CLI Check

Skip if `oc whoami` returns a username.

If `oc` is not found:
> "Install the OpenShift CLI: https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html"

If `oc` is found but not logged in:
> "Log in to your OpenShift cluster: oc login <OCP_API_URL> --token=<OCP_TOKEN>
> Your OCP API URL and token are on the RHDP Services page."

Validate after login:
```bash
oc whoami && echo "✅ oc — logged in" || echo "❌ oc — not logged in"
```

## Step 5 — helm CLI Check

Skip if `helm version` succeeds.

If missing:
> "Install helm: https://helm.sh/docs/intro/install/"

Validate:
```bash
helm version --short && echo "✅ helm" || echo "❌ helm — not found"
```

## Step 6 — Final Validation

```bash
grep -q "galaxy_server.rh_certified" ~/.ansible/ansible.cfg 2>/dev/null && \
  grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "✅ ~/.ansible/ansible.cfg" || echo "❌ ~/.ansible/ansible.cfg"

test -d ~/.ansible/collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform" || echo "❌ ansible.platform"
test -d ~/.ansible/collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller" || echo "❌ ansible.controller"
test -d ~/.ansible/collections/ansible_collections/kubernetes/core && \
  echo "✅ kubernetes.core" || echo "❌ kubernetes.core"

oc whoami 2>/dev/null && echo "✅ oc — logged in" || echo "❌ oc — not logged in"
helm version --short 2>/dev/null && echo "✅ helm" || echo "❌ helm"
```

If everything is green:
```
All prerequisites are in place.
Ready to run /selfservice-bootstrap.
```

If anything is red, tell the user which step to re-run. Do not proceed until all items are green.
