# snow_to_infoblox

Read a ServiceNow catalog task, find a CSV attachment in **NIOS-native CSV import format**, and submit it to Infoblox via the WAPI `csv_import` endpoint.

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
- infoblox.nios_modules    # provider/auth helpers; csv_import uses uri directly
```

Install with `ansible-galaxy collection install -r collections/requirements.yml`.

## Inputs

| Variable | Source | Required | Default |
|---|---|---|---|
| `snow_task_sys_id` | survey / extra-vars | yes | — |
| `infoblox_csv_operation` | survey / extra-vars | no | `MERGE` |
| `csv_poll_retries` | extra-vars | no | `60` |
| `csv_poll_delay_seconds` | extra-vars | no | `5` |

### `infoblox_csv_operation` choices

| Operation | Behaviour |
|---|---|
| `INSERT` | Only create new records; fail rows whose key already exists |
| `MERGE` | Update fields present in CSV, leave others untouched (default) |
| `OVERRIDE` | Replace entire records with CSV contents |
| `CUSTOM` | Use per-row `IMPORT-ACTION` column to drive behaviour |
| `REPLACE` | Bulk-replace all records of this type |
| `DELETE` | Remove records named in CSV |

## What it does

1. **Preflight** — assert all required env vars are set; validate `snow_task_sys_id` and `infoblox_csv_operation`.
2. **Fetch sc_task** — `servicenow.itsm.api` GET by `sys_id`.
3. **Locate CSV** — list `sys_attachment`, filter for `*.csv`, pick newest.
4. **Download CSV** — `ansible.builtin.uri` against `/api/now/attachment/<sys_id>/file`.
5. **NIOS upload** — 3-step WAPI fileop dance:
   - `POST /wapi/v.../fileop?_function=uploadinit` → token + one-time URL
   - `POST` the CSV (multipart) to that URL with the session cookie
   - `POST /wapi/v.../fileop?_function=csv_import` with the token + operation
6. **Poll** — GET the returned `csvimporttask` reference until status is one of `COMPLETED`, `COMPLETED_WITH_ERRORS`, `FAILED`, `STOPPED`.
7. **Update sc_task** — write a summary including `lines_processed`, `lines_succeeded`, `errors_count`, the NIOS `import_id`, and a pointer to the NIOS CSV Job Manager. Close as `Closed Complete` only on full success.
8. **Cleanup** — remove the staged CSV from `/tmp`.

## Expected CSV format

The CSV must use **Infoblox NIOS native CSV import format**. See [`examples/sample_hosts.csv`](../../examples/sample_hosts.csv):

```csv
HEADER-HostRecord,configure_for_dns,FQDN,ADDRESSES
HostRecord,True,web01.example.com,10.20.30.11
HostRecord,True,api01.example.com,"10.20.40.21,10.20.40.22"
```

The first column on the `HEADER-` line is the object type (`HEADER-HostRecord` for host records; other examples: `HEADER-NetworkContainer`, `HEADER-Aaaa`, `HEADER-Cname`). Data rows start with the bare object type literal. NIOS handles all column-mapping and validation server-side — the role does not parse the CSV.

## Local testing

You don't need AAP to run this — just export the env vars yourself:

```bash
# ServiceNow
export SNOW_INSTANCE="mycompany.service-now.com"
export SNOW_USER="ansible_svc"
export SNOW_PASSWORD="..."

# Infoblox
export INFOBLOX_HOST="grid-master.example.com"
export INFOBLOX_USER="ansible_svc"
export INFOBLOX_PASSWORD="..."
export INFOBLOX_WAPI_VERSION="2.11"

ansible-galaxy collection install -r collections/requirements.yml

ansible-playbook playbooks/snow_to_infoblox.yml \
  -e snow_task_sys_id=abcdef0123456789abcdef0123456789 \
  -e infoblox_csv_operation=MERGE
```

## Failure modes

| Condition | Behaviour |
|---|---|
| Any required env var missing | Preflight fails with full list of missing vars |
| `snow_task_sys_id` not 32 chars | Preflight fails with the supplied value |
| `infoblox_csv_operation` not in valid set | Preflight fails listing valid options |
| sc_task not found | Asserts and fails |
| No CSV attachment on the task | Asserts and fails |
| Downloaded CSV is empty | Asserts and fails |
| `csv_import` returns non-201 | `uri` task fails with the WAPI error body |
| csvimporttask never reaches terminal status | `until` loop exhausts retries → fail |
| csvimporttask ends `FAILED`/`STOPPED` | sc_task gets work_notes; play hard-fails |
| Some rows failed but at least one succeeded | sc_task gets work_notes; play passes (partial-success allowed) |

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
ansible-playbook --syntax-check playbooks/snow_to_infoblox.yml
```
