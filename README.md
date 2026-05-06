# infoblox-snow

ServiceNow → Infoblox CSV import automation, packaged as an Ansible role for AAP.

```
.
├── ansible.cfg
├── aap/
│   ├── credential_types/
│   │   ├── servicenow.yml      # AAP custom credential type definition
│   │   └── infoblox.yml        # AAP custom credential type definition
│   └── job_template.yml        # Example job template wiring
├── collections/
│   └── requirements.yml        # servicenow.itsm + community.general + infoblox.nios_modules
├── playbooks/
│   └── snow_to_infoblox.yml    # Entry playbook (called by AAP)
└── roles/
    └── snow_to_infoblox/       # The role itself — see roles/snow_to_infoblox/README.md
```

**Credentials are NEVER stored in this repo** — neither plaintext nor `ansible-vault`.
AAP injects them via custom credential types; locally you `export` them in your shell.

See [`roles/snow_to_infoblox/README.md`](roles/snow_to_infoblox/README.md) for the role docs.

## First-time AAP setup

1. Apply the credential types (`aap/credential_types/*.yml`).
2. Create one Credential of each type, populated with real values.
3. Create the project, point it at this repo.
4. Apply the job template (`aap/job_template.yml`), attaching both credentials.
5. Launch — survey will prompt for the sc_task `sys_id`.

## Sample data

A representative CSV is at [`examples/sample_hosts.csv`](examples/sample_hosts.csv) — 10 rows using the default column names (`hostname`, `ip_address`, `dns_view`, `comment`). Attach it to a sandbox sc_task to exercise the full flow, or use it as a column-layout reference when shaping the catalog item form in ServiceNow.

## Local quickstart

```bash
ansible-galaxy collection install -r collections/requirements.yml

export SNOW_INSTANCE=mycompany.service-now.com SNOW_USER=svc SNOW_PASSWORD=...
export INFOBLOX_HOST=gm.example.com INFOBLOX_USER=svc INFOBLOX_PASSWORD=...

ansible-playbook playbooks/snow_to_infoblox.yml \
  -e snow_task_sys_id=<32-char-sys_id>
```
