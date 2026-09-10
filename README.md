# elk_cluster_post_os_patching_restart

Ansible rolling restart for a self-managed Elastic Stack after monthly OS patching.

https://github.com/nwlterry/elk_cluster_post_os_patching_restart

Automates the official Elasticsearch rolling-restart procedure, then rolls Fleet Server, APM on RHEL, Logstash, and Kibana one host at a time. Two APM servers run as OpenShift `elastic-agent` containers and are **health-checked only** (no OS patch, no reboot, no systemd).

**Azure DevOps Server wiki source (paste-ready):** [docs/AZURE_DEVOPS_WIKI.md](docs/AZURE_DEVOPS_WIKI.md)

Reference: [Full-cluster restart and rolling restart procedures](https://www.elastic.co/docs/deploy-manage/maintenance/start-stop-services/full-cluster-restart-rolling-restart-procedures)

## Restart sequence

`3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana`

| Step | Group | Count | Action |
|------|-------|-------|--------|
| 1 | `es_masters` | 3 | Reboot/restart ES |
| 2 | `es_data_hot` | 6 | Allocation disable, reboot/restart ES, wait green |
| 3 | `es_data_cold` | 5 | Same as hot |
| 4 | `es_ml` | 2 | Reboot/restart ES; jobs paused via `_ml/set_upgrade_mode` |
| 5 | `fleet` | 2 | Reboot/restart `elastic-agent` |
| 6 | `apm_rhel` | 2 | Reboot/restart `elastic-agent` on RHEL VMs |
| 7 | `apm_openshift` | 2 | Port/HTTP check only — no OS patch |
| 8 | `logstash` | 2 | Reboot/restart Logstash |
| 9 | `kibana` | 2 | Reboot/restart Kibana |

`--tags apm` runs both APM plays. `--tags apm_rhel` or `--tags apm_openshift` selects one side.

## Current cluster

| Role | Inventory group | Count | systemd unit (default) | Port check |
|------|-----------------|-------|------------------------|------------|
| Dedicated master | `es_masters` | 3 | `elasticsearch` | 9200 |
| Data hot | `es_data_hot` | 6 | `elasticsearch` | 9200 |
| Data cold | `es_data_cold` | 5 | `elasticsearch` | 9200 |
| ML | `es_ml` | 2 | `elasticsearch` | 9200 |
| Fleet Server | `fleet` | 2 | `elastic-agent` | 8220 |
| APM RHEL VM | `apm_rhel` | 2 | `elastic-agent` | 8200 |
| APM OpenShift container | `apm_openshift` | 2 | n/a — check only | 8200 |
| Logstash | `logstash` | 2 | `logstash` | 9600 |
| Kibana | `kibana` | 2 | `kibana` | 5601 |
| **Total** | | **16 ES + 10 edge = 26** | | |

Parent group `apm` = `apm_rhel` + `apm_openshift` (4). Precheck fails if inventory sizes or live `_cluster/health.number_of_nodes` (ES only = 16) do not match. Counts live in `group_vars/all.yml` (`expected_topology`).

## How host information is collected

The playbook does **not** discover nodes from Elasticsearch and reboot whatever it finds. The reboot list is the static inventory. Elasticsearch is used only to **validate** those names and health.

| Field | File | Purpose |
|-------|------|--------|
| `inventory_hostname` | `inventories/production.yml` key | Ansible target name |
| `ansible_host` | same file | SSH target for VMs; TCP/HTTP target for port checks |
| `es_node_name` | same file, ES hosts only | Must equal Elasticsearch `node.name` in `_cat/nodes` |
| `es_api_host` | `group_vars/all.yml` | Cluster API used for all health/allocation/ML calls |
| `expected_topology` / `expected_es_nodes` | `group_vars/all.yml` | Counts the precheck asserts |
| `skip_restart` / `ansible_connection: local` | `apm_openshift` | No SSH into OpenShift pods |

`ansible.cfg` sets `inventory = inventories/production.yml`.

## Layout

```
.
├── ansible.cfg
├── inventories/production.yml
├── group_vars/all.yml
├── group_vars/vault.yml.example
├── playbooks/rolling_restart.yml
├── playbooks/tasks/
│   ├── restart_es_node.yml
│   └── restart_edge_node.yml
└── docs/AZURE_DEVOPS_WIKI.md
```

## Before first run

1. Fill in `inventories/production.yml`. `es_node_name` must match `node.name`.
2. APM RHEL: real VM IPs, SSH + `elastic-agent` enabled.
3. APM OpenShift: `ansible_host` = Service or Route on `apm_port`. Group already has `skip_restart: true` and `ansible_connection: local`.
4. Set `es_api_host` in `group_vars/all.yml`.
5. Vault-encrypt `vault_es_api_password`.
6. Snapshot before the run.

## Run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml -e reboot_host=false --ask-vault-pass

ansible-playbook playbooks/rolling_restart.yml --tags apm --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags apm_rhel --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags apm_openshift --ask-vault-pass
```

## What the playbook does not do

- It does **not** apply OS patches.
- It does **not** take snapshots.
- It does **not** use the node shutdown API.
- It does **not** discover hosts from `_cat/nodes` or from the OpenShift API.
- It does **not** restart OpenShift APM pods.
- Fleet / APM HTTP status checks treat `401` as up.

## Recovery if the playbook stops mid-way

1. Check `_cluster/health` and `_cluster/settings`.
2. If `cluster.routing.allocation.enable` is still `primaries`, clear it to `null`.
3. If ML jobs are paused: `POST _ml/set_upgrade_mode?enabled=false`
4. Re-run with `--tags hot --limit es-hot-04` or `--tags apm_rhel --limit apm-02`.
