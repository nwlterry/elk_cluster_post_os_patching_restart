# elk_cluster_post_os_patching_restart

Ansible rolling **host reboot** after monthly OS patching. Node order follows Elastic official **upgrade rolling restart** (same version; no package upgrade in this playbook).

- Playbook: https://github.com/nwlterry/elk_cluster_post_os_patching_restart
- Rules: https://github.com/nwlterry/elk_stack_monitor
- ADO wiki: [docs/AZURE_DEVOPS_WIKI.md](docs/AZURE_DEVOPS_WIKI.md)
- Rule objects: [docs/objects/](docs/objects/)

Official references:

- ES node order: https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/elasticsearch
- Stack order: https://www.elastic.co/docs/deploy-manage/upgrade/plan-upgrade
- Per-node restart (no flush used here): https://www.elastic.co/docs/deploy-manage/maintenance/start-stop-services/full-cluster-restart-rolling-restart-procedures

Indexing stays on. **No `POST /_flush`.** RHEL hosts reboot. OpenShift APM is check-only.

## Official sequence used here

Elastic ES order: data tiers frozen → cold → warm → hot → other data → ML/ingest/coordinating → **masters last**.  
This cluster has no frozen/warm tier.

Elastic stack order after ES: **Kibana** → Fleet Server and APM → ingest (Logstash).

`mute alerts → 5 cold → 6 hot → 2 ML → 3 masters → 2 Kibana → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → unmute alerts`

| Step | Group | Count | Action |
|------|-------|-------|--------|
| Pre | localhost | — | Green + 16 ES nodes; mute cluster-status rules; no flush |
| 1 | `es_data_cold` | 5 | Allocation `primaries`, **reboot host**, wait green |
| 2 | `es_data_hot` | 6 | Same as cold |
| 3 | `es_ml` | 2 | **Reboot host**; ML upgrade_mode |
| 4 | `es_masters` | 3 | **Reboot host** (masters last) |
| 5 | `kibana` | 2 | **Reboot host** (after ES) |
| 6 | `fleet` | 2 | **Reboot host** |
| 7 | `apm_rhel` | 2 | **Reboot host** |
| 8 | `apm_openshift` | 2 | Check only |
| 9 | `logstash` | 2 | **Reboot host** (ingest last) |
| Post | localhost | — | Green + unmute rules |

## Required objects

| Kind | Object |
|------|--------|
| Playbook | `playbooks/rolling_restart.yml` |
| Tasks | `restart_es_node.yml`, `restart_edge_node.yml`, `mute_stack_monitoring.yml`, `unmute_stack_monitoring.yml` |
| Inventory | `inventories/production.yml` |
| Vars | `group_vars/all.yml` |
| Wiki | `docs/AZURE_DEVOPS_WIKI.md` |
| Rule (informational) | `docs/objects/elk_cluster_status-informational.json` → `CC \| Elasticsearch \| Cluster Status ( Informational )` |
| Rule (warning / red) | `docs/objects/elk_cluster_status-warning.json` → `DevOps \| Elasticsearch \| Cluster Status ( Warning )` |

Mute by exact Kibana name or tag `maintenance-mute`.

## Run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

## Recovery

1. Clear `allocation.enable` if stuck on `primaries`.
2. `POST _ml/set_upgrade_mode?enabled=false`
3. `--tags unmute`
4. Resume e.g. `--tags cold --limit es-cold-03`.
