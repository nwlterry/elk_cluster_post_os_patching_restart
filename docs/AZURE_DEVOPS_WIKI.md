# ELK cluster post-OS-patching rolling restart

**Repository:** [nwlterry/elk_cluster_post_os_patching_restart](https://github.com/nwlterry/elk_cluster_post_os_patching_restart)  
**Stack monitoring:** [nwlterry/elk_stack_monitor](https://github.com/nwlterry/elk_stack_monitor)  
**Playbook:** `playbooks/rolling_restart.yml`  
**Inventory:** `inventories/production.yml`

This is a **same-version OS-patch rolling host reboot**. It is not an Elasticsearch major-version upgrade.

**Indexing stays on.** No `POST /_flush`. `allocation.enable=primaries` only blocks replica relocation.

**OS patching requires a host reboot** on every RHEL VM. OpenShift APM containers are check-only.

**Stack monitoring:** before the first reboot the playbook disables the two cluster-status rules exported as `elk_cluster_status-informational.json` and `elk_cluster_status-warning.json`. After the cluster is green with 16 ES nodes, those rules are enabled again.

---

## Restart sequence (authoritative)

One host at a time (`serial: 1`). `reboot_host: true` on every RHEL play.

| Step | Play | Group | Count | Action |
|------|------|-------|-------|--------|
| Pre | Pre-checks | localhost | — | Green, 16 ES nodes, mute cluster-status rules. No flush. |
| 1/9 | Masters | `es_masters` | 3 | **Reboot host** |
| 2/9 | Data hot | `es_data_hot` | 6 | Allocation `primaries` → **reboot host** → rejoin → allocation on → green |
| 3/9 | Data cold | `es_data_cold` | 5 | Same as hot |
| 4/9 | ML | `es_ml` | 2 | **Reboot host** (jobs in upgrade_mode) |
| 5/9 | Fleet | `fleet` | 2 | **Reboot host**, check :8220 |
| 6/9 | APM RHEL | `apm_rhel` | 2 | **Reboot host**, check :8200 |
| 7/9 | APM OpenShift | `apm_openshift` | 2 | Check only — no OS patch |
| 8/9 | Logstash | `logstash` | 2 | **Reboot host**, check :9600 |
| 9/9 | Kibana | `kibana` | 2 | **Reboot host**, check :5601 |
| Post | Post-checks | localhost | — | Allocation on, ML jobs on, green, 16 nodes |
| Post | Unmute | localhost | — | Re-enable cluster-status rules |

**Compact order:**

`mute alerts → 3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana → unmute alerts`

---

## Stack monitoring alerts

Source: https://github.com/nwlterry/elk_stack_monitor

| File | Kibana rule name | Fires when |
|------|------------------|------------|
| `elk_cluster_status-informational.json` | `CC \| Elasticsearch \| Cluster Status ( Informational )` | Query count on `.ds-metrics-elasticsearch.stack_monitoring.cluster_stats-default*` |
| `elk_cluster_status-warning.json` | `DevOps \| Elasticsearch \| Cluster Status ( Warning )` | `elasticsearch.cluster.stats.status` is **red** (last 5m) |

Match method: exact name in `stack_monitor_rule_names` **or** tag `maintenance-mute`.

Kibana API (`kibana_api_host` in `group_vars/all.yml`):

```text
GET  {kibana}/api/alerting/rules/_find
POST {kibana}/api/alerting/rule/{id}/_disable    # precheck
POST {kibana}/api/alerting/rule/{id}/_enable     # postcheck / --tags unmute
```

If a run is aborted:

```bash
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

Set `kibana_api_host` to a Kibana VIP or instance that is reachable after play 9/9.

---

## Topology

16 ES + 10 edge = 26 inventory entries. Parent `apm` = 4 (2 RHEL + 2 OpenShift).

---

## How to run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

---

## Recovery if the run stops mid-way

1. `_cluster/health` and `_cluster/settings`. If allocation is still `primaries`, set it to `null`.
2. `POST _ml/set_upgrade_mode?enabled=false`
3. `--tags unmute` so the cluster-status rules are not left disabled.
4. Resume with `--tags hot --limit es-hot-04` (etc.).
