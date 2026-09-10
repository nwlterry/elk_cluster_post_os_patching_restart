# elk_cluster_post_os_patching_restart

Ansible rolling **host reboot** after monthly OS patching.

- Playbook repo: https://github.com/nwlterry/elk_cluster_post_os_patching_restart
- Rule repo: https://github.com/nwlterry/elk_stack_monitor
- **ADO wiki page (paste this):** [docs/AZURE_DEVOPS_WIKI.md](docs/AZURE_DEVOPS_WIKI.md)
- **Rule objects stored with that page:** [docs/objects/](docs/objects/)

Indexing stays on. No `POST /_flush`. RHEL hosts reboot. OpenShift APM is check-only.

## Required objects

| Kind | Object |
|------|--------|
| Playbook | `playbooks/rolling_restart.yml` |
| Tasks | `restart_es_node.yml`, `restart_edge_node.yml`, `mute_stack_monitoring.yml`, `unmute_stack_monitoring.yml` |
| Inventory | `inventories/production.yml` |
| Vars | `group_vars/all.yml` |
| Wiki | `docs/AZURE_DEVOPS_WIKI.md` |
| Kibana rule (informational) | `docs/objects/elk_cluster_status-informational.json` |
| Kibana rule (warning / red) | `docs/objects/elk_cluster_status-warning.json` |

Same two JSON files exist in `elk_stack_monitor`.

### Kibana names that must match

| File | Exact Kibana name |
|------|-------------------|
| `elk_cluster_status-informational.json` | `CC \| Elasticsearch \| Cluster Status ( Informational )` |
| `elk_cluster_status-warning.json` | `DevOps \| Elasticsearch \| Cluster Status ( Warning )` |

Mute by that name **or** tag `maintenance-mute`.

## Restart sequence

`mute alerts → 3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana → unmute alerts`

| Step | Group | Count | Action |
|------|-------|-------|--------|
| Pre | localhost | — | Green + 16 ES nodes; mute the two cluster-status rules |
| 1 | `es_masters` | 3 | **Reboot host** |
| 2 | `es_data_hot` | 6 | Allocation disable, **reboot host**, wait green |
| 3 | `es_data_cold` | 5 | Same as hot |
| 4 | `es_ml` | 2 | **Reboot host**; ML upgrade_mode |
| 5 | `fleet` | 2 | **Reboot host** |
| 6 | `apm_rhel` | 2 | **Reboot host** |
| 7 | `apm_openshift` | 2 | Check only |
| 8 | `logstash` | 2 | **Reboot host** |
| 9 | `kibana` | 2 | **Reboot host** |
| Post | localhost | — | Green + unmute rules |

16 ES + 10 edge = 26 inventory entries. Parent `apm` = 4.

## Run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

Set `es_api_host` and `kibana_api_host` in `group_vars/all.yml`.

## ADO Wiki import

1. Wiki → New page → title `ELK cluster post-OS-patching rolling restart`.
2. Paste [docs/AZURE_DEVOPS_WIKI.md](docs/AZURE_DEVOPS_WIKI.md).
3. Attach both files under `docs/objects/` to that page (or publish `docs/` as a code wiki).

## Recovery

1. Clear `cluster.routing.allocation.enable` if it is stuck on `primaries`.
2. `POST _ml/set_upgrade_mode?enabled=false`
3. `--tags unmute`
4. Resume with `--tags hot --limit es-hot-04` (etc.).
