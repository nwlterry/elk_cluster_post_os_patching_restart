# ELK cluster post-OS-patching rolling restart

Paste this page into Azure DevOps Server Wiki. Keep objects under `docs/objects/` with this page.

**GitHub (playbook):** https://github.com/nwlterry/elk_cluster_post_os_patching_restart  
**GitHub (rules):** https://github.com/nwlterry/elk_stack_monitor  
**Playbook:** `playbooks/rolling_restart.yml`  
**Inventory:** `inventories/production.yml`  
**Vars:** `group_vars/all.yml`

Same-version OS-patch **host reboot**, not an ES version upgrade. Node **order** follows Elastic official upgrade rolling restart.

Official docs:

- ES node order: https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/elasticsearch
- Stack order: https://www.elastic.co/docs/deploy-manage/upgrade/plan-upgrade
- Per-node rolling restart: https://www.elastic.co/docs/deploy-manage/maintenance/start-stop-services/full-cluster-restart-rolling-restart-procedures

- Indexing stays on. **No `POST /_flush`.** No stop-indexing.
- Data nodes: `allocation.enable=primaries` while the host is down, then `null` after it rejoins.
- RHEL VMs reboot. OpenShift APM is check-only.
- Mute the two cluster-status rules before the first reboot; unmute after green + 16 ES nodes.

---

## Official order applied to this cluster

Elastic ES order: frozen → cold → warm → hot → other data → ML/ingest/coordinating → **masters last**.  
This cluster has no frozen or warm tier.

Elastic stack order after ES: **Kibana** → Fleet Server and APM → ingest (Logstash).

`mute alerts → 5 cold → 6 hot → 2 ML → 3 masters → 2 Kibana → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → unmute alerts`

| Step | Play | Group | Count | Action |
|------|------|-------|-------|--------|
| Pre | Pre-checks | localhost | — | Green, 16 ES nodes, mute rules. **No flush.** |
| 1/9 | Data cold | `es_data_cold` | 5 | Allocation `primaries` → **reboot** → rejoin → allocation on → green |
| 2/9 | Data hot | `es_data_hot` | 6 | Same as cold |
| 3/9 | ML | `es_ml` | 2 | **Reboot** (non-master non-data) |
| 4/9 | Masters | `es_masters` | 3 | **Reboot** (masters last) |
| 5/9 | Kibana | `kibana` | 2 | **Reboot** after ES |
| 6/9 | Fleet | `fleet` | 2 | **Reboot**, check :8220 |
| 7/9 | APM RHEL | `apm_rhel` | 2 | **Reboot**, check :8200 |
| 8/9 | APM OpenShift | `apm_openshift` | 2 | Check only |
| 9/9 | Logstash | `logstash` | 2 | **Reboot** (ingest last), check :9600 |
| Post | Post-checks | localhost | — | Allocation on, ML jobs on, green, 16 nodes |
| Post | Unmute | localhost | — | Re-enable cluster-status rules |

---

## Required objects (store with this wiki page)

| Object | Path |
|--------|------|
| Playbook | `playbooks/rolling_restart.yml` |
| ES node tasks | `playbooks/tasks/restart_es_node.yml` |
| Edge node tasks | `playbooks/tasks/restart_edge_node.yml` |
| Mute / unmute | `playbooks/tasks/mute_stack_monitoring.yml`, `unmute_stack_monitoring.yml` |
| Inventory | `inventories/production.yml` |
| Group vars | `group_vars/all.yml` |
| Informational rule | `docs/objects/elk_cluster_status-informational.json` → `CC \| Elasticsearch \| Cluster Status ( Informational )` |
| Warning rule (red) | `docs/objects/elk_cluster_status-warning.json` → `DevOps \| Elasticsearch \| Cluster Status ( Warning )` |

Both rules tagged `maintenance-mute`. Index: `.ds-metrics-elasticsearch.stack_monitoring.cluster_stats-default*`.

---

## Topology

16 ES + 10 edge = 26. Parent `apm` = 4 (2 RHEL + 2 OpenShift).

---

## Mute / unmute

```text
GET  {kibana}/api/alerting/rules/_find
POST {kibana}/api/alerting/rule/{id}/_disable
POST {kibana}/api/alerting/rule/{id}/_enable
```

```bash
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

---

## How to run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
```

---

## Recovery

1. If allocation is still `primaries`, set it to `null`.
2. `POST _ml/set_upgrade_mode?enabled=false`
3. `--tags unmute`
4. Resume e.g. `--tags cold --limit es-cold-03`.

---

## Import this page into Azure DevOps Server Wiki

1. Wiki → New page → title `ELK cluster post-OS-patching rolling restart`.
2. Paste this markdown.
3. Attach both files in `docs/objects/` (or publish `docs/` as a code wiki).
