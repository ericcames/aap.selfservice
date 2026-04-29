# AAP Self-Service First-Time Setup

This skill walks a new user through every local prerequisite needed to run `/aap-bootstrap` in this repo. Check what already exists, guide creation of anything missing, and end with a full preflight validation.

Run this once on a new machine. If you've already done this setup, skip straight to `/selfservice-bootstrap`.

## Orientation

Print this once at the start:

```
You're setting up prerequisites to deploy the AAP Self-Service Portal from your laptop.
This takes about 10 minutes the first time.

We'll set up:
  1. Your Automation Hub API token (~/.ansible/ansible.cfg)
  2. Your vault password file (~/.ansible/secrets2)
  3. Required Ansible collections (ansible.platform, ansible.controller, kubernetes.core)
  4. oc CLI — logged in to OpenShift
  5. helm CLI — available locally
  6. Your SSH key pair (local key + public key hosted at a URL)
  7. Your vault file (secrets the demo uses at runtime)
  8. Your identity defaults (~/.ansible/aap_defaults.yml)

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

# secrets2
test -s ~/.ansible/secrets2 && echo "EXISTS: secrets2" || echo "MISSING: secrets2"

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

# SSH key
test -f ~/.ssh/id_rsa && echo "EXISTS: ~/.ssh/id_rsa" || echo "MISSING: ~/.ssh/id_rsa"

# user defaults
python3 -c "
import yaml, os
path = os.path.expanduser('~/.ansible/aap_defaults.yml')
try:
    d = yaml.safe_load(open(path)) or {}
    for k in ['my_vault','my_remote_vault','my_remote_ssh_pub_key','my_windows_catalog_short_description']:
        v = d.get(k,'')
        print(('EXISTS: ' if v else 'MISSING: ') + k + (': ' + str(v) if v else ''))
except FileNotFoundError:
    print('MISSING: ~/.ansible/aap_defaults.yml')
"
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

## Step 3 — secrets2 Setup

Skip if `~/.ansible/secrets2` exists and is non-empty.

Explain:
```
~/.ansible/secrets2 contains one line: your vault password.
This is used to decrypt your vault file at runtime.
```

Prompt the user for their vault password, then write it as a single line:

```bash
mkdir -p ~/.ansible
touch ~/.ansible/secrets2
chmod 600 ~/.ansible/secrets2
```

Confirm the file is non-empty after writing.

## Step 4 — Collections Install

Skip if all three collections are found by `ansible-galaxy collection list` (they may be at `~/.ansible/collections/` or the system Python path).

Show this command and ask for confirmation before running:

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

## Step 5 — oc CLI Check

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

## Step 6 — helm CLI Check

Skip if `helm version` succeeds.

If missing:
> "Install helm: https://helm.sh/docs/intro/install/"

Validate:
```bash
helm version --short && echo "✅ helm" || echo "❌ helm — not found"
```

## Step 7 — SSH Key Check

The SSH private key goes into your vault credential in AAP. The public key must be hosted at a public HTTPS raw URL so it can be fetched at runtime.

### 7a. Check for a local RSA key

```bash
test -f ~/.ssh/id_rsa && echo "EXISTS" || echo "MISSING"
```

If missing, offer to generate one:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
```
Show the command and ask for confirmation. Note that AWS requires RSA keys.

### 7b. Check for a hosted public key URL

Ask:
> "Is your public SSH key hosted at a public HTTPS raw URL?
> (e.g. https://raw.githubusercontent.com/youruser/yourrepo/main/id_rsa.pub)"

If not, direct the user to host it in a public GitHub repo using the raw URL format.

### 7c. Validate the hosted URL

```bash
curl -s "<hosted_pub_key_url>" | head -1
```

Confirm the response starts with `ssh-rsa`. If it returns HTML or a redirect, the URL is wrong.

### 7d. Verify key pair matches

```bash
DERIVED=$(ssh-keygen -y -f ~/.ssh/id_rsa 2>/dev/null)
HOSTED=$(curl -s "<hosted_pub_key_url>" | awk '{print $1, $2}')
DERIVED_MATERIAL=$(echo "$DERIVED" | awk '{print $1, $2}')

if [ "$DERIVED_MATERIAL" = "$HOSTED" ]; then
  echo "✅ Key pair matches"
else
  echo "❌ Key pair mismatch — hosted key does not match ~/.ssh/id_rsa"
fi
```

If `ssh-keygen -y` fails (passphrase-protected key), fall back to comparing `~/.ssh/id_rsa.pub` against the hosted URL.

## Step 8 — Vault File Check

The vault file holds runtime secrets. It must be hosted at a public HTTPS raw URL.

Ask:
> "Do you have a vault file hosted at a public HTTPS raw URL?"

If yes, ask for the URL and validate:
```bash
curl -s -L "<vault_url>" | head -5
```
Check that the response is YAML (not HTML) and returns HTTP 200.

If no, direct the user to:
> "Copy the example vault from https://github.com/ericcames/sourcefiles/blob/main/vault_example.yml,
> fill in your values, encrypt it with ansible-vault, and host it in a public GitHub repo at a raw URL."

## Step 9 — User Identity Defaults

Write to `~/.ansible/aap_defaults.yml`. Skip any key that already has a non-empty value.

Ask for each missing value:

- **`my_vault`** — name for your vault credential in AAP (e.g. `Eric Ames`)
- **`my_windows_catalog_short_description`** — ServiceNow Windows catalog description (default: `AAP Windows AWS Daily Demo`)
- **`my_remote_vault`** — URL from Step 8
- **`my_remote_ssh_pub_key`** — URL from Step 7b

Write/merge using Python:

```bash
python3 << 'EOF'
import yaml, os

path = os.path.expanduser('~/.ansible/aap_defaults.yml')
updates = {
    'my_vault': '<value>',
    'my_windows_catalog_short_description': '<value>',
    'my_remote_vault': '<url>',
    'my_remote_ssh_pub_key': '<url>',
}

try:
    existing = yaml.safe_load(open(path)) or {}
except FileNotFoundError:
    existing = {}

existing.update({k: v for k, v in updates.items() if v})
with open(path, 'w') as f:
    yaml.dump(existing, f, default_flow_style=False)
print('✅ ~/.ansible/aap_defaults.yml written')
EOF
```

## Step 10 — Final Validation

```bash
grep -q "galaxy_server.rh_certified" ~/.ansible/ansible.cfg 2>/dev/null && \
  grep -v "PASTE_YOUR_TOKEN_HERE" ~/.ansible/ansible.cfg | grep -q "token=" && \
  echo "✅ ~/.ansible/ansible.cfg" || echo "❌ ~/.ansible/ansible.cfg"

test -s ~/.ansible/secrets2 && echo "✅ ~/.ansible/secrets2" || echo "❌ ~/.ansible/secrets2"

test -d ~/.ansible/collections/ansible_collections/ansible/platform && \
  echo "✅ ansible.platform" || echo "❌ ansible.platform"
test -d ~/.ansible/collections/ansible_collections/ansible/controller && \
  echo "✅ ansible.controller" || echo "❌ ansible.controller"
test -d ~/.ansible/collections/ansible_collections/kubernetes/core && \
  echo "✅ kubernetes.core" || echo "❌ kubernetes.core"

oc whoami 2>/dev/null && echo "✅ oc — logged in" || echo "❌ oc — not logged in"
helm version --short 2>/dev/null && echo "✅ helm" || echo "❌ helm"

test -f ~/.ssh/id_rsa && echo "✅ ~/.ssh/id_rsa" || echo "❌ ~/.ssh/id_rsa"
test -s ~/.ansible/aap_defaults.yml && echo "✅ ~/.ansible/aap_defaults.yml" || echo "❌ ~/.ansible/aap_defaults.yml"
```

If everything is green:
```
All prerequisites are in place.
Ready to run /aap-bootstrap.
```

If anything is red, tell the user which step to re-run. Do not proceed until all items are green.
