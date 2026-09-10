# ELK cluster post-OS-patching rolling restart

Paste this page into Azure DevOps Server Wiki (New page). Attach or keep the objects under `docs/objects/` with this page.

**GitHub (playbook):** https://github.com/nwlterry/elk_cluster_post_os_patching_restart  
**GitHub (rules):** https://github.com/nwlterry/elk_stack_monitor  
**Playbook:** `playbooks/rolling_restart.yml`  
**Inventory:** `inventories/production.yml`  
**Vars:** `group_vars/all.yml`

This is a **same-version OS-patch rolling host reboot**, not an Elasticsearch version upgrade.

- Indexing stays on. No `POST /_flush`.
- `cluster.routing.allocation.enable=primaries` only blocks replica relocation. Primaries keep writes.
- Every RHEL VM is **rebooted**. OpenShift APM is check-only.
- Before reboot, Kibana rules below are **disabled**. After 16 ES nodes are green they are **enabled**.

---

## Required objects (store with this wiki page)

### Playbook repo files

| Object | Path |
|--------|------|
| Playbook | `playbooks/rolling_restart.yml` |
| ES node tasks | `playbooks/tasks/restart_es_node.yml` |
| Edge node tasks | `playbooks/tasks/restart_edge_node.yml` |
| Mute alerts | `playbooks/tasks/mute_stack_monitoring.yml` |
| Unmute alerts | `playbooks/tasks/unmute_stack_monitoring.yml` |
| Inventory | `inventories/production.yml` |
| Group vars | `group_vars/all.yml` |
| Vault example | `group_vars/vault.yml.example` |
| Ansible config | `ansible.cfg` |

### Kibana rule objects (copies in this repo)

| Wiki / repo file | Kibana rule name (exact) | Tags |
|------------------|--------------------------|------|
| `docs/objects/elk_cluster_status-informational.json` | `CC \| Elasticsearch \| Cluster Status ( Informational )` | Elasticsearch, Monitoring, Infrastructure, Informational, **maintenance-mute** |
| `docs/objects/elk_cluster_status-warning.json` | `DevOps \| Elasticsearch \| Cluster Status ( Warning )` | Elasticsearch, Monitoring, Infrastructure, Warning, **maintenance-mute** |

Canonical copies also live in `elk_stack_monitor` with the same file names.

Mute match: exact name in `stack_monitor_rule_names` **or** tag `maintenance-mute`.

Index pattern used by both rules:

`.ds-metrics-elasticsearch.stack_monitoring.cluster_stats-default*`

---

## Restart sequence (authoritative)

One host at a time (`serial: 1`). RHEL plays set `reboot_host: true`.

| Step | Play | Group | Count | Action |
|------|------|-------|-------|--------|
| Pre | Pre-checks | localhost | — | Green, 16 ES nodes, mute the two cluster-status rules. No flush. |
| 1/9 | Masters | `es_masters` | 3 | **Reboot host** |
| 2/9 | Data hot | `es_data_hot` | 6 | Allocation `primaries` → **reboot host** → rejoin → allocation on → green |
| 3/9 | Data cold | `es_data_cold` | 5 | Same as hot |
| 4/9 | ML | `es_ml` | 2 | **Reboot host** (ML `upgrade_mode`) |
| 5/9 | Fleet | `fleet` | 2 | **Reboot host**, check :8220 |
| 6/9 | APM RHEL | `apm_rhel` | 2 | **Reboot host**, check :8200 |
| 7/9 | APM OpenShift | `apm_openshift` | 2 | Check only — no OS patch |
| 8/9 | Logstash | `logstash` | 2 | **Reboot host**, check :9600 |
| 9/9 | Kibana | `kibana` | 2 | **Reboot host**, check :5601 |
| Post | Post-checks | localhost | — | Allocation on, ML jobs on, green, 16 nodes |
| Post | Unmute | localhost | — | Re-enable the two cluster-status rules |

Compact:

`mute alerts → 3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana → unmute alerts`

Ansible tags: `precheck`, `elasticsearch`, `masters`, `hot`, `cold`, `ml`, `edge`, `fleet`, `apm`, `apm_rhel`, `apm_openshift`, `logstash`, `kibana`, `postcheck`, `mute`, `unmute`, `monitoring`.

---

## Topology objects

| Role | Inventory group | Count | Runtime | Port | Reboot |
|------|-----------------|-------|---------|------|--------|
| Dedicated master | `es_masters` | 3 | `elasticsearch` RHEL | 9200 | yes |
| Data hot | `es_data_hot` | 6 | `elasticsearch` RHEL | 9200 | yes |
| Data cold | `es_data_cold` | 5 | `elasticsearch` RHEL | 9200 | yes |
| ML | `es_ml` | 2 | `elasticsearch` RHEL | 9200 | yes |
| Fleet Server | `fleet` | 2 | `elastic-agent` RHEL | 8220 | yes |
| APM RHEL | `apm_rhel` | 2 | `elastic-agent` RHEL | 8200 | yes |
| APM OpenShift | `apm_openshift` | 2 | `elastic-agent` container | 8200 | no |
| Logstash | `logstash` | 2 | `logstash` RHEL | 9600 | yes |
| Kibana | `kibana` | 2 | `kibana` RHEL | 5601 | yes |
| **Total** | | **16 ES + 10 edge = 26** | | | |

Parent `apm` = `apm_rhel` + `apm_openshift` (4).

`expected_topology` / `expected_es_nodes: 16` / `expected_edge_nodes: 10` / `expected_total_hosts: 26` live in `group_vars/all.yml`.

---

## How host information is collected

Hosts are **not** discovered from Elasticsearch or OpenShift. The reboot list is the static inventory. ES is used only to validate health and names.

| Field | Source | Use |
|-------|--------|-----|
| `inventory_hostname` | key in `inventories/production.yml` | Ansible name |
| `ansible_host` | same file | SSH to VMs; TCP/HTTP target |
| `es_node_name` | ES hosts only | must equal `node.name` in `GET /_cat/nodes` |
| `es_api_host` | `group_vars/all.yml` | cluster health / allocation / ML |
| `kibana_api_host` | `group_vars/all.yml` | mute / unmute rules |
| `skip_restart` + `ansible_connection: local` | `apm_openshift` | no SSH into pods |

---

## Mute / unmute API

```text
GET  {kibana}/api/alerting/rules/_find
POST {kibana}/api/alerting/rule/{id}/_disable
POST {kibana}/api/alerting/rule/{id}/_enable
```

`stack_monitor_rule_names` in `group_vars/all.yml`:

- `CC | Elasticsearch | Cluster Status ( Informational )`
- `DevOps | Elasticsearch | Cluster Status ( Warning )`

If a run is aborted:

```bash
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

---

## How to run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

---

## Recovery

1. `_cluster/health` and `_cluster/settings`. If allocation is still `primaries`, set it to `null`.
2. `POST _ml/set_upgrade_mode?enabled=false`
3. `--tags unmute` so the two cluster-status rules are not left disabled.
4. Resume with a tag and limit, for example `--tags hot --limit es-hot-04`.

---

## Rule object: informational

File: `docs/objects/elk_cluster_status-informational.json`

```json
{
  "name": "CC | Elasticsearch | Cluster Status ( Informational )",
  "tags": [
    "Elasticsearch",
    "Monitoring",
    "Infrastructure",
    "Informational",
    "maintenance-mute"
  ],
  "schedule": { "interval": "1m" },
  "params": {
    "timeField": "@timestamp",
    "index": [".ds-metrics-elasticsearch.stack_monitoring.cluster_stats-default*"],
    "esQuery": "{ \"query\": { \"match_all\": {} } }",
    "size": 100,
    "thresholdComparator": ">",
    "timeWindowSize": 5,
    "timeWindowUnit": "m",
    "threshold": [5],
    "aggType": "count",
    "groupBy": "all",
    "searchType": "esQuery"
  },
  "actions": [],
  "alert_delay": { "active": 1 }
}
```

---

## Rule object: warning (cluster red)

File: `docs/objects/elk_cluster_status-warning.json`

```json
{
  "name": "DevOps | Elasticsearch | Cluster Status ( Warning )",
  "tags": [
    "Elasticsearch",
    "Monitoring",
    "Infrastructure",
    "Warning",
    "maintenance-mute"
  ],
  "schedule": { "interval": "1m" },
  "params": {
    "searchType": "esQuery",
    "timeWindowSize": 5,
    "timeWindowUnit": "m",
    "threshold": [1],
    "thresholdComparator": ">",
    "index": [".ds-metrics-elasticsearch.stack_monitoring.cluster_stats-default*"],
    "timeField": "@timestamp",
    "esQuery": "{ \"query\": { \"bool\": { \"filter\": [ { \"range\": { \"@timestamp\": { \"gte\": \"now-5m\", \"lte\": \"now\" } } }, { \"match\": { \"elasticsearch.cluster.stats.status\": \"red\" } } ] } } }"
  },
  "actions": [],
  "alert_delay": { "active": 1 }
}
```

Import into Kibana → Stack Management → Rules. Names must stay exact. Tag `maintenance-mute` must stay.

---

## Import this page into Azure DevOps Server Wiki

1. Project → Wiki → New page.
2. Title: `ELK cluster post-OS-patching rolling restart`.
3. Paste this markdown.
4. Attach `docs/objects/elk_cluster_status-informational.json` and `docs/objects/elk_cluster_status-warning.json` to the page (or publish the Git `docs/` folder as a code wiki).
