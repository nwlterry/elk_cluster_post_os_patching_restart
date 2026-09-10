# elk_cluster_post_os_patching_restart

Ansible rolling **host reboot** for a self-managed Elastic Stack after monthly OS patching.

https://github.com/nwlterry/elk_cluster_post_os_patching_restart

Stack monitoring rules live in https://github.com/nwlterry/elk_stack_monitor (node-down + disk → Slack). This playbook **disables** those Kibana rules before the first reboot and **enables** them again after all 16 ES nodes are green.

Two APM servers run as OpenShift `elastic-agent` containers and are **health-checked only** (no OS patch, no reboot).

**Azure DevOps Server wiki source:** [docs/AZURE_DEVOPS_WIKI.md](docs/AZURE_DEVOPS_WIKI.md)

Indexing stays on. No `POST /_flush`.

## Restart sequence

`mute alerts → 3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana → unmute alerts`

| Step | Group | Count | Action |
|------|-------|-------|--------|
| Pre | localhost | — | Green + 16 ES nodes; mute `elk_stack_monitor` / `maintenance-mute` rules |
| 1 | `es_masters` | 3 | **Reboot host** |
| 2 | `es_data_hot` | 6 | Allocation disable, **reboot host**, wait green |
| 3 | `es_data_cold` | 5 | Same as hot |
| 4 | `es_ml` | 2 | **Reboot host**; jobs paused via `_ml/set_upgrade_mode` |
| 5 | `fleet` | 2 | **Reboot host** (`elastic-agent`) |
| 6 | `apm_rhel` | 2 | **Reboot host** (`elastic-agent`) |
| 7 | `apm_openshift` | 2 | Port/HTTP check only — no OS patch |
| 8 | `logstash` | 2 | **Reboot host** |
| 9 | `kibana` | 2 | **Reboot host** |
| Post | localhost | — | Allocation on, ML jobs on, unmute alerts |

Every RHEL play sets `reboot_host: true` because OS patching requires a reboot. OpenShift APM is the only exception (`skip_restart: true`).

## Stack monitoring

Rules from [elk_stack_monitor](https://github.com/nwlterry/elk_stack_monitor):

- `Elasticsearch Node Monitoring Alert` (disk > 85%)
- `Elasticsearch Node Down Alert` (cardinality of `elasticsearch.node.name` < 16)
- Optional built-ins: Elasticsearch nodes changed, Missing monitoring data, Elasticsearch cluster status

Matched by exact name (`stack_monitor_rule_names`) or tag `maintenance-mute`.

```bash
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

Set `kibana_api_host` in `group_vars/all.yml` to a Kibana URL that stays reachable (or comes back after play 9/9).

## Current cluster

| Role | Inventory group | Count | Action |
|------|-----------------|-------|--------|
| Dedicated master | `es_masters` | 3 | reboot |
| Data hot | `es_data_hot` | 6 | reboot |
| Data cold | `es_data_cold` | 5 | reboot |
| ML | `es_ml` | 2 | reboot |
| Fleet Server | `fleet` | 2 | reboot |
| APM RHEL VM | `apm_rhel` | 2 | reboot |
| APM OpenShift container | `apm_openshift` | 2 | check only |
| Logstash | `logstash` | 2 | reboot |
| Kibana | `kibana` | 2 | reboot |
| **Total** | | **16 ES + 10 edge = 26** | |

## Run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

## Recovery if the playbook stops mid-way

1. Check `_cluster/health` and `_cluster/settings`. Clear `allocation.enable` if stuck on `primaries`.
2. `POST _ml/set_upgrade_mode?enabled=false` if jobs are paused.
3. Re-enable alerts: `--tags unmute`.
4. Resume with `--tags hot --limit es-hot-04` (and so on).
