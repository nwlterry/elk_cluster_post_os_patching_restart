# ELK cluster post-OS-patching rolling restart

**Repository:** [nwlterry/elk_cluster_post_os_patching_restart](https://github.com/nwlterry/elk_cluster_post_os_patching_restart)  
**Playbook:** `playbooks/rolling_restart.yml`  
**Inventory:** `inventories/production.yml`  
**Elastic reference:** [Full-cluster restart and rolling restart procedures](https://www.elastic.co/docs/deploy-manage/maintenance/start-stop-services/full-cluster-restart-rolling-restart-procedures)

This page describes the **same-version OS-patch rolling restart** for the production Elastic Stack. It is **not** an Elasticsearch major-version upgrade.

OS patches are applied by a separate process. This playbook only restarts (or health-checks) stack components in a safe order.

---

## Restart sequence (authoritative)

One host at a time (`serial: 1`, `order: inventory`).

| Step | Play | Inventory group | Count | Action |
|------|------|-----------------|-------|--------|
| Pre | Pre-checks | localhost | — | Cluster must be **green**, inventory counts must match, live ES node count must be **16**. Optional: enable ML `upgrade_mode`, flush indices. |
| 1/9 | Dedicated masters | `es_masters` | 3 | Reboot host or restart `elasticsearch`. No allocation disable. |
| 2/9 | Data hot | `es_data_hot` | 6 | `allocation.enable=primaries` → reboot/restart → wait node in `_cat/nodes` → `allocation.enable=null` → wait **green** + no relocating/initializing. |
| 3/9 | Data cold | `es_data_cold` | 5 | Same as hot. |
| 4/9 | ML | `es_ml` | 2 | Reboot/restart `elasticsearch`. Jobs already paused via `_ml/set_upgrade_mode`. |
| 5/9 | Fleet Server | `fleet` | 2 | Reboot/restart `elastic-agent`. Wait port **8220** + HTTP (200 or 401). |
| 6/9 | APM RHEL | `apm_rhel` | 2 | Reboot/restart `elastic-agent` on RHEL VMs. Wait port **8200** + HTTP (200 or 401). |
| 7/9 | APM OpenShift | `apm_openshift` | 2 | **No restart.** Containers are not OS-patched. Port/HTTP check only from the controller. |
| 8/9 | Logstash | `logstash` | 2 | Reboot/restart `logstash`. Wait port **9600**. |
| 9/9 | Kibana | `kibana` | 2 | Reboot/restart `kibana`. Wait port **5601**. |
| Post | Post-checks | localhost | — | Force allocation back on, disable ML `upgrade_mode`, assert **green**, 0 unassigned, 16 ES nodes. |

**Compact order:**

`3 masters → 6 hot → 5 cold → 2 ML → 2 Fleet → 2 APM RHEL → 2 APM OpenShift (check only) → 2 Logstash → 2 Kibana`

Tags:

- `--tags apm` — both APM plays
- `--tags apm_rhel` — RHEL VMs only
- `--tags apm_openshift` — OpenShift check only

---

## Topology

| Role | Group | Count | Unit / runtime | Port |
|------|-------|-------|----------------|------|
| Dedicated master | `es_masters` | 3 | `elasticsearch` on RHEL | 9200 |
| Data hot | `es_data_hot` | 6 | `elasticsearch` on RHEL | 9200 |
| Data cold | `es_data_cold` | 5 | `elasticsearch` on RHEL | 9200 |
| ML | `es_ml` | 2 | `elasticsearch` on RHEL | 9200 |
| Fleet Server | `fleet` | 2 | `elastic-agent` on RHEL | 8220 |
| APM (RHEL VM) | `apm_rhel` | 2 | `elastic-agent` on RHEL | 8200 |
| APM (OpenShift) | `apm_openshift` | 2 | `elastic-agent` container | 8200 |
| Logstash | `logstash` | 2 | `logstash` on RHEL | 9600 |
| Kibana | `kibana` | 2 | `kibana` on RHEL | 5601 |
| **Total** | | **16 ES + 10 edge = 26** | | |

Parent group `apm` = `apm_rhel` + `apm_openshift` (**4** APM servers).

Expected counts live in `group_vars/all.yml`:

- `expected_es_nodes: 16`
- `expected_edge_nodes: 10`
- `expected_total_hosts: 26`
- `expected_topology.apm: 4`, `apm_rhel: 2`, `apm_openshift: 2`

---

## How host information is collected

Hosts are **not** auto-discovered from Elasticsearch or OpenShift. The playbook reads a **static inventory**, then uses the ES API only to **validate** health and that ES names came back.

| Field | Source | Used for |
|-------|--------|----------|
| `inventory_hostname` | Key in `inventories/production.yml` | Ansible target name |
| `ansible_host` | Same file | SSH to RHEL VMs; TCP/HTTP target for port checks |
| `es_node_name` | Same file, ES hosts only | Must equal Elasticsearch `node.name` in `GET /_cat/nodes` |
| `es_api_host` | `group_vars/all.yml` | All cluster health / allocation / ML API calls |
| `skip_restart` + `ansible_connection: local` | `apm_openshift` group | No SSH and no systemd on OpenShift pods |

`ansible.cfg` sets `inventory = inventories/production.yml`.

Live checks:

1. `GET /_cluster/health` on `es_api_host`
2. `GET /_cat/nodes` — live ES names; rejoin wait intersects this list with `[es_node_name, inventory_hostname]`
3. Inventory group sizes vs `expected_topology`
4. Live ES node count must equal **16** (edge hosts are not in this number)

If `node.name` in `elasticsearch.yml` does not match `es_node_name`, the VM reboots but the playbook waits on `_cat/nodes` until timeout.

OpenShift APM is not queried via `oc`. Set `ansible_host` to the Service or Route that answers on port 8200.

```text
inventory YAML  -->  groups + ansible_host + es_node_name
        |
        v
precheck (localhost)
  GET _cluster/health + GET _cat/nodes   vs   expected_topology
        |
        v
serial 1 per group
  ES RHEL     SSH ansible_host -> reboot/restart elasticsearch
              localhost: wait TCP 9200 on ansible_host
              localhost: wait es_node_name in _cat/nodes
              data nodes: allocation primaries -> null + green
  Fleet       SSH + elastic-agent + :8220
  APM RHEL    SSH + elastic-agent + :8200
  APM OCP     localhost TCP/HTTP :8200 only (skip_restart)
  Logstash    SSH + logstash + :9600
  Kibana      SSH + kibana + :5601
        |
        v
postcheck  allocation on, ML jobs on, 16 ES nodes, green
```

---

## Per-node Elasticsearch loop (hot and cold)

1. Wait cluster **green**.
2. `PUT /_cluster/settings` persistent `cluster.routing.allocation.enable: primaries`.
3. Reboot the host (`reboot_host: true`) or `systemctl restart elasticsearch`.
4. Wait SSH (if rebooted).
5. Wait TCP 9200 on that `ansible_host`.
6. Poll `_cat/nodes` until `es_node_name` appears.
7. Set `cluster.routing.allocation.enable: null`.
8. Wait **green** and no relocating / initializing shards.
9. Next host.

Masters and ML skip steps 2 and 7 (no allocation disable).

---

## OpenShift APM vs RHEL APM

| | APM RHEL (`apm_rhel`) | APM OpenShift (`apm_openshift`) |
|--|------------------------|----------------------------------|
| Runtime | `elastic-agent` on RHEL VM | `elastic-agent` container |
| OS patching | Yes — included in monthly patch | No |
| Ansible connection | SSH | `local` (controller) |
| `skip_restart` | false | true |
| Reboot / systemd | Yes | No |
| What the playbook does | Restart agent, then check :8200 | Check Service/Route :8200 only |

OpenShift **liveness / readiness / startup probes** own container restart. This playbook must not `oc delete pod` or `systemctl` those containers.

---

## How to run

```bash
cd elk_cluster_post_os_patching_restart

ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml -e reboot_host=false --ask-vault-pass
```

`--tags precheck` is also tagged `always`. To skip precheck on a single tier after the cluster is already green: `--skip-tags always`.

---

## Prerequisites

1. OS patches already applied on RHEL VMs (not on OpenShift APM).
2. Recent Elasticsearch snapshot.
3. Inventory hostnames / IPs / Routes filled in.
4. `es_node_name` matches `node.name`.
5. `es_api_host` is a VIP or a master that stays reachable.
6. Vault password for `vault_es_api_password`.
7. Units enabled: `elasticsearch`, `elastic-agent` (Fleet + APM RHEL), `logstash`, `kibana`.
8. Disk below the ES low watermark on nodes that will restart.

---

## What this playbook does not do

- Apply OS patches
- Take snapshots
- Call `PUT _nodes/{id}/shutdown` (ECE/ECK API)
- Discover hosts from `_cat/nodes` or the OpenShift API
- Restart OpenShift APM pods

Fleet and APM HTTP checks treat **401** as “process is up” when the status path is auth-gated.

---

## Recovery if the run stops mid-way

1. Check `_cluster/health` and `_cluster/settings`.
2. If allocation is still `primaries`:

```http
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

3. If ML jobs stay paused: `POST _ml/set_upgrade_mode?enabled=false`
4. Resume with a tag and limit, for example:

```bash
ansible-playbook playbooks/rolling_restart.yml --tags hot --limit es-hot-04 --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags apm_rhel --limit apm-02 --ask-vault-pass
```

---

## Repo layout

```text
ansible.cfg
inventories/production.yml
group_vars/all.yml
group_vars/vault.yml.example
playbooks/rolling_restart.yml
playbooks/tasks/restart_es_node.yml
playbooks/tasks/restart_edge_node.yml
docs/AZURE_DEVOPS_WIKI.md
```
