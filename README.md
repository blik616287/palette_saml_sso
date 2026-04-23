# Palette <-> JumpCloud SAML SSO Automation

Ansible playbook that fully automates the SAML SSO setup between a Spectro Cloud Palette tenant and JumpCloud.

All resource names are prefixed with the tenant name so multiple tenants can be configured side-by-side without collision.

## Quickstart

```bash
cd ansible/palette_saml_sso

# 1. Create your local env file from the template and fill in the 3 required secrets.
cp sso-env.sh.example sso-env.sh
chmod 600 sso-env.sh
$EDITOR sso-env.sh
#   JUMPCLOUD_API_KEY          (jca_...)
#   PALETTE_SYSTEM_PASSWORD    (terraform output palette_admin_password)
#   PALETTE_TENANT_PASSWORD    (password for PALETTE_TENANT_ADMIN)

# 2. Source the env.
source sso-env.sh

# 3. Run the playbook. Defaults target the "default" tenant.
ansible-playbook playbook.yml

# Run against another tenant — override via env:
PALETTE_TENANT_NAME=ISC ansible-playbook playbook.yml

# Tear down what the playbook created for a given tenant (scoped, safe):
PALETTE_TENANT_NAME=ISC ansible-playbook teardown.yml
```

`sso-env.sh` is gitignored so real credentials won't get committed. The `sso-env.sh.example` template in the repo shows every variable the playbook consumes, with comments for every optional override.

## What it does

1. **Palette tenant ensure**: Logs in as system admin, creates the tenant if missing, reactivates it if soft-deleted (API + MongoDB fallback).
2. **Palette tenant auth**: Logs in as tenant admin and fetches the current SAML config.
3. **JumpCloud app**: Finds or creates a Custom SAML 2.0 app labelled `{tenant}-Palette` with:
   - IdP URL: `https://sso.jumpcloud.com/saml2/{tenant-slug}-palette`
   - SP Entity ID and ACS URL matching Palette's
   - NameID = user email, format `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`
   - Attributes: `Email`, `FirstName`, `LastName`, and `SpectroTeam` group attribute
   - **No SP certificate** (avoids JumpCloud validating the non-spec-compliant signed AuthnRequest Palette emits on HTTP-Redirect binding)
   - Activates the app
4. **JumpCloud user group**: Finds or creates the `{tenant}-SpectroTeam` user group (tagged with a per-tenant marker) and associates it with the app.
5. **JumpCloud group members**: Adds each email listed in `jumpcloud_group_members` to the group (idempotent — no-op if the list is empty).
6. **Palette SAML config**: Uploads the JumpCloud IdP metadata to Palette, enables SSO, forces `syncSsoTeams=true` (via MongoDB fallback, since the API silently ignores this field).
7. **Palette team**: Creates a Palette team named `{tenant}-SpectroTeam` with `Tenant Admin` role so SAML-authenticated users inherit the role.

## Why "no SP certificate" in JumpCloud

Palette signs AuthnRequests but delivers them via HTTP-Redirect binding while embedding the signature in the XML (not in URL query params as the spec requires). JumpCloud with an SP cert uploaded validates signatures strictly and rejects Palette's malformed request with `internal error`. With no SP cert, JumpCloud's metadata advertises `WantAuthnRequestsSigned="false"` and it accepts Palette's AuthnRequest as-is.

## Prerequisites

- JumpCloud admin API key (`jca_...`)
- Palette system admin password (for tenant create/activate)
- Palette tenant admin credentials (used after the tenant exists)
- `aws` and `kubectl` on PATH with access to the Palette EKS cluster (only used by the MongoDB fallback paths: tenant un-soft-delete and `syncSsoTeams` fix)

## Layout

```
ansible/palette_saml_sso/
  ansible.cfg                        # points at ./inventory
  playbook.yml                       # create/update SSO config (entrypoint)
  teardown.yml                       # tear down SSO config (scoped, safe)
  inventory/
    hosts.yml                        # localhost only
    group_vars/
      all.yml                        # every variable with env-var fallbacks
  tasks/
    palette_tenant_ensure.yml        # system admin: create + activate tenant
    palette_auth.yml                 # tenant admin login + fetch SAML config
    jumpcloud_app.yml                # create/find SAML app, activate, get IdP metadata
    jumpcloud_group.yml              # create/find user group with marker + associate
    jumpcloud_group_members.yml      # add users to the group by email
    palette_saml_config.yml          # push IdP metadata + force syncSsoTeams + create team
  README.md
```

## Configuration

Every variable lives in `inventory/group_vars/all.yml`. Edit that file for persistent per-environment defaults, or override any value at runtime by setting the matching env var of the same uppercased name.

| Variable | Env var | Default | Purpose |
|---|---|---|---|
| `palette_tenant_name` | `PALETTE_TENANT_NAME` | `default` | Tenant org name — drives every generated resource name |
| `jumpcloud_api_key` | `JUMPCLOUD_API_KEY` | — | JumpCloud admin API key (`jca_...`), **required** |
| `jumpcloud_group_members` | — | `[marty@insightsoftmax.net]` | YAML list of emails to add to the group. Set to `[]` to skip membership management |
| `palette_base_url` | `PALETTE_BASE_URL` | `https://palette.isc-spectro-dev.click` | Palette system URL |
| `palette_system_password` | `PALETTE_SYSTEM_PASSWORD` | — | System admin password, **required** |
| `palette_tenant_admin` | `PALETTE_TENANT_ADMIN` | `marty@insightsoftmax.com` | Tenant admin email |
| `palette_tenant_password` | `PALETTE_TENANT_PASSWORD` | — | Tenant admin password, **required** |
| `palette_team_role_name` | `PALETTE_TEAM_ROLE_NAME` | `Tenant Admin` | Role assigned to the auto-created team |
| `palette_eks_cluster_name` | `PALETTE_EKS_CLUSTER_NAME` | `spectroihmeae-ue2-palette` | EKS cluster for the MongoDB fallback paths |
| `palette_aws_region` | `AWS_REGION` | `us-east-2` | AWS region for `aws eks update-kubeconfig` |
| `palette_aws_profile` | `AWS_PROFILE` | `spectro` | AWS profile for `aws eks update-kubeconfig` |
| `palette_skip_mongo_sync_fix` | `PALETTE_SKIP_MONGO_SYNC_FIX` | `false` | Skip the MongoDB `syncSsoTeams=true` update |

### Derived names

All driven by `palette_tenant_name` (a `palette_tenant_slug = palette_tenant_name | lower` is used where Palette's router is case-sensitive):

| Resource | Value |
|---|---|
| Tenant URL | `https://{slug}.palette.<base>` |
| ACS URL | `https://palette.<base>/v1/auth/org/{slug}/saml/callback` |
| JumpCloud IdP URL | `https://sso.jumpcloud.com/saml2/{slug}-palette` |
| JumpCloud app label | `{tenant}-Palette` |
| JumpCloud group | `{tenant}-SpectroTeam` |
| Palette team | `{tenant}-SpectroTeam` |
| Group marker (for safe teardown) | `Managed by palette-saml-sso playbook (tenant: {slug})` |

## Run

```bash
cd ansible/palette_saml_sso

# Secrets via env (or set them in group_vars/all.yml, but prefer env/vault for
# credentials so they aren't committed).
export JUMPCLOUD_API_KEY='jca_...'
export PALETTE_SYSTEM_PASSWORD='...'
export PALETTE_TENANT_PASSWORD='...'

# Defaults target the "default" tenant. Override with env:
# PALETTE_TENANT_NAME=ISC ansible-playbook playbook.yml

ansible-playbook playbook.yml
```

## After running

1. Users in `jumpcloud_group_members` see the `{tenant}-Palette` tile in the JumpCloud user portal — clicking it performs IdP-initiated SSO into Palette.
2. Visiting `https://{slug}.palette.<base>` also works (SP-initiated, subject to Palette's AuthnRequest limitations).
3. First login auto-provisions the user into Palette and places them in the `{tenant}-SpectroTeam` team (`Tenant Admin` role).

Re-running the playbook is safe — existing JumpCloud apps, groups, and Palette teams are reused (looked up by name).

## Scope guarantees

This playbook is **strictly scoped** to one tenant's SAML integration.

**On the JumpCloud side:**
- Creates/finds the SAML application whose `ssoUrl` is exactly `https://sso.jumpcloud.com/saml2/{slug}-palette`
- Creates the `{tenant}-SpectroTeam` user group if missing, tagged with the per-tenant marker description
- Associates that group with the SAML app
- Adds users to the group only if listed in `jumpcloud_group_members`
- On teardown: deletes only user groups that (a) are associated with the deleted app **and** (b) have the exact per-tenant marker. Pre-existing groups (no marker, or with a different tenant's marker) are preserved — only disassociated from the app.

**On the Palette side:**
- Creates/activates the target tenant if needed
- Updates that tenant's SAML config (only that tenant)
- Creates/updates the `{tenant}-SpectroTeam` team in that tenant

It will **never**:
- Touch any JumpCloud app or group not belonging to this tenant's integration
- Create, modify, or delete any JumpCloud user
- Touch any other Palette tenant
- Modify Palette system-wide settings (other than creating/activating the named tenant)

## Full clean-room test

```bash
cd ansible/palette_saml_sso

export JUMPCLOUD_API_KEY='jca_...'
export PALETTE_SYSTEM_PASSWORD="$(cd ../../../ && terraform output -raw palette_admin_password)"
export PALETTE_TENANT_PASSWORD='...'
export PALETTE_TENANT_NAME='default'

# Tear down (deletes the tenant's SAML app + marked groups, clears Palette SAML config, deletes the team)
ansible-playbook teardown.yml

# Re-create everything
ansible-playbook playbook.yml

# Re-run to confirm idempotency
ansible-playbook playbook.yml
```

Then verify in an incognito browser: log in to JumpCloud → click the `{tenant}-Palette` tile → lands in Palette.
