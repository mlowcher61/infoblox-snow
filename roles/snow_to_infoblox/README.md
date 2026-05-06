# snow_to_infoblox

Read a ServiceNow catalog task, find a CSV attachment, parse it, and import the records into Infoblox NIOS as host records.

Designed for **Ansible Automation Platform (AAP)** with **custom credential types** — no `ansible-vault`, no secrets in git.

## How credentials work

Two AAP custom credential types ship in [`aap/credential_types/`](../../aap/credential_types/):

| Credential type | Injects env vars |
|---|---|
| `ServiceNow ITSM` | `SNOW_INSTANCE`, `SNOW_USER`, `SNOW_PASSWORD`, `SNOW_AUTH_TYPE`, `SNOW_CLIENT_ID`, `SNOW_CLIENT_SECRET` |
| `Infoblox NIOS` | `INFOBLOX_HOST`, `INFOBLOX_USER`, `INFOBLOX_PASSWORD`, `INFOBLOX_WAPI_VERSION` |

The role reads them with `lookup('env', ...)` in [`defaults/main.yml`](defaults/main.yml). Override per run via the AAP survey or `-e var=value`.

## Required collections

```yaml
- servicenow.itsm
- community.general
- infoblox.nios_modules
```

Install with `ansible-galaxy collection install -r collections/requirements.yml`.

## Inputs

| Variable | Source | Required | Default |
|---|---|---|---|
| `snow_task_sys_id` | survey / extra-vars | yes | — |
| `csv_col_hostname` | survey / extra-vars | no | `hostname` |
| `csv_col_ip` | survey / extra-vars | no | `ip_address` |
| `csv_col_dns_view` | survey / extra-vars | no | `dns_view` |
| `csv_col_comment` | survey / extra-vars | no | `comment` |
| `infoblox_default_view` | extra-vars | no | `default` |

## What it does

1. **Preflight** — assert all required env vars are set; build the SNOW provider block.
2. **Fetch sc_task** — `servicenow.itsm.api` GET by `sys_id`.
3. **Locate CSV** — list `sys_attachment`, filter for `*.csv`, pick newest.
4. **Download CSV** — `ansible.builtin.uri` against `/api/now/attachment/<sys_id>/file`.
5. **Parse CSV** — `community.general.read_csv`, verify required columns.
6. **Import** — loop `infoblox.nios_modules.nios_host_record` with per-row block/rescue.
7. **Update sc_task** — PATCH `work_notes` with summary, set state to `Closed Complete` (3) on partial-or-better success.
8. **Cleanup** — remove the staged CSV from `/tmp`.

## Local testing

You don't need AAP to run this — just export the env vars yourself:

```bash
# ServiceNow
export SNOW_INSTANCE="mycompany.service-now.com"
export SNOW_USER="ansible_svc"
export SNOW_PASSWORD="..."
export SNOW_AUTH_TYPE="basic"   # or 'oauth' (then also export CLIENT_ID/SECRET)

# Infoblox
export INFOBLOX_HOST="grid-master.example.com"
export INFOBLOX_USER="ansible_svc"
export INFOBLOX_PASSWORD="..."
export INFOBLOX_WAPI_VERSION="2.11"

ansible-galaxy collection install -r collections/requirements.yml

ansible-playbook playbooks/snow_to_infoblox.yml \
  -e snow_task_sys_id=abcdef0123456789abcdef0123456789
```

## Failure modes

| Condition | Behaviour |
|---|---|
| Any required env var missing | Preflight fails with full list of missing vars |
| `snow_task_sys_id` not 32 chars | Preflight fails with the supplied value |
| sc_task not found | Asserts and fails |
| No CSV attachment on the task | Asserts and fails |
| CSV present but empty / missing columns | Asserts and fails |
| Single Infoblox row fails | Logged to `import_failures`, loop continues |
| **All** Infoblox rows fail | Job ends red after the SNOW work_notes update |

## Why no `ansible-vault`

| | `ansible-vault` | AAP custom credentials |
|---|---|---|
| Secrets in git | encrypted blob | nothing — git is clean |
| Local testing | `--ask-vault-pass` | `export VAR=value` |
| Rotation | edit, re-encrypt, commit, deploy | update credential object in AAP |
| Audit trail | git history | AAP credential audit log |

## Testing

```bash
ansible-lint
yamllint .
molecule test -s default   # if molecule is installed
```
