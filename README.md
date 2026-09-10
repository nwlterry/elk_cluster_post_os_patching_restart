# elk_cluster_post_os_patching_restart

Ansible rolling restart for a self-managed Elastic Stack after monthly OS patching.

https://github.com/nwlterry/elk_cluster_post_os_patching_restart

Automates the official Elasticsearch rolling-restart procedure, then rolls Fleet Server, Logstash, and Kibana one host at a time. Both APM servers run as OpenShift containers and are **health-checked only** (no reboot, no systemd).

Reference: [Full-cluster restart and rolling restart procedures](https://www.elastic.co/docs/deploy-manage/maintenance/start-stop-services/full-cluster-restart-rolling-restart-procedures)

## Current cluster

| Role | Inventory group | Count | systemd unit (default) | Port check |
|------|-----------------|-------|------------------------|------------|
| Dedicated master | `es_masters` | 3 | `elasticsearch` | 9200 |
| Data hot | `es_data_hot` | 6 | `elasticsearch` | 9200 |
| Data cold | `es_data_cold` | 5 | `elasticsearch` | 9200 |
| ML | `es_ml` | 2 | `elasticsearch` | 9200 |
| Fleet Server | `fleet` | 2 | `elastic-agent` | 8220 |
| APM (OpenShift container) | `apm` | 2 | n/a — check only | 8200 |
| Logstash | `logstash` | 2 | `logstash` | 9600 |
| Kibana | `kibana` | 2 | `kibana` | 5601 |
| **Total** | | **16 ES + 8 edge = 24** | | |

Precheck fails if inventory sizes or live `_cluster/health.number_of_nodes` (ES only = 16) do not match. Counts live in `group_vars/all.yml` (`expected_topology`).

## How host information is collected

The playbook does **not** discover nodes from Elasticsearch and reboot whatever it finds. The reboot list is the static inventory. Elasticsearch is used only to **validate** those names and health.

| Field | File | Purpose |
|-------|------|--------|
| `inventory_hostname` | `inventories/production.yml` key | Ansible target name (`es-hot-01`, `apm-01`, …) |
| `ansible_host` | same file | SSH target for VMs; TCP/HTTP target for port checks |
| `es_node_name` | same file, ES hosts only | Must equal Elasticsearch `node.name` in `_cat/nodes` |
| `es_api_host` | `group_vars/all.yml` | Cluster API used for all health/allocation/ML calls |
| `expected_topology` / `expected_es_nodes` | `group_vars/all.yml` | Counts the precheck asserts against inventory and live ES |
| `skip_restart` / `ansible_connection: local` | `apm` group | No SSH into OpenShift pods; check Service/Route only |

`ansible.cfg` sets `inventory = inventories/production.yml`.

Live collection during precheck and each ES node:

1. `GET /_cluster/health` on `es_api_host` — status, node counts, unassigned/relocating/initializing.
2. `GET /_cat/nodes?h=name,...` — live ES names. Rejoin wait intersects this list with `[es_node_name, inventory_hostname]`.
3. Inventory group lengths vs `expected_topology`.
4. `number_of_nodes` and `_cat/nodes` length must equal `expected_es_nodes` (16). Edge hosts are not in that number.

If `node.name` in `elasticsearch.yml` does not match `es_node_name`, the host reboots but the `_cat/nodes` wait times out.

APM is not looked up via `oc` or `_cat/nodes`. Set `ansible_host` to the OpenShift Service or Route that answers on `apm_port`.

```
inventory YAML  ──►  groups + ansible_host + es_node_name
        │
        ▼
precheck (localhost)  ──  _cluster/health + _cat/nodes  vs  expected_topology
        │
        ▼
serial: 1 per group, order: inventory
  ES     SSH ansible_host → reboot/restart
         localhost wait TCP on ansible_host
         localhost wait es_node_name in _cat/nodes
         data nodes: allocation primaries → null + green
  Fleet / Logstash / Kibana  SSH + own port/HTTP
  APM    localhost port/HTTP only (skip_restart)
        │
        ▼
postcheck  allocation on, ML jobs on, 16 ES nodes, green
```

## Node order and per-node flow

Same-version OS-patch restart uses dedicated-master-first so quorum is stable before data nodes move. (Elastic *version-upgrade* order is data tiers first and masters last; this repo is not that path.)

| Step | Group | Count | Allocation disable | Action |
|------|-------|-------|--------------------|--------|
| 1 | `es_masters` | 3 | no | Reboot/restart ES |
| 2 | `es_data_hot` | 6 | yes | Reboot/restart ES |
| 3 | `es_data_cold` | 5 | yes | Reboot/restart ES |
| 4 | `es_ml` | 2 | no | Reboot/restart ES; jobs paused via `_ml/set_upgrade_mode` |
| 5 | `fleet` | 2 | n/a | Reboot/restart `elastic-agent` |
| 6 | `apm` | 2 | n/a | OpenShift — port/HTTP check only |
| 7 | `logstash` | 2 | n/a | Reboot/restart Logstash |
| 8 | `kibana` | 2 | n/a | Reboot/restart Kibana |

### Precheck (`hosts: localhost`)

1. Cluster must be green (`require_initial_green`).
2. Optional fail on `_cat/pending_tasks`.
3. Print `_cat/nodes`.
4. Assert inventory sizes and 16 live ES nodes.
5. Optional `POST /_ml/set_upgrade_mode?enabled=true`.
6. Optional `POST /_flush`.

### Each Elasticsearch node (`tasks/restart_es_node.yml`)

1. Wait green on the cluster API.
2. Data nodes only: `cluster.routing.allocation.enable = primaries`.
3. Reboot the host (`reboot_host: true`) or `systemctl restart elasticsearch`.
4. Wait SSH if rebooted.
5. Wait TCP `es_http_port` on `ansible_host` from localhost.
6. Poll `_cat/nodes` until `es_node_name` appears.
7. Data nodes only: `allocation.enable = null`.
8. Wait green and no relocating/initializing shards.
9. Next host (`serial: 1`).

### Each edge node (`tasks/restart_edge_node.yml`)

1. Wait ES green.
2. If `skip_restart` (APM): skip reboot and systemd.
3. Else reboot or restart `edge_service`.
4. Wait service port and optional HTTP status from localhost (`200` or `401` for Fleet/APM).

### Postcheck (`hosts: localhost`)

1. Force `allocation.enable: null`.
2. `POST /_ml/set_upgrade_mode?enabled=false`.
3. Final health + `_cat/nodes`.
4. Fail unless green, 0 unassigned, 16 ES nodes.

## Layout

```
.
├── ansible.cfg
├── inventories/production.yml          # edit hostnames / IPs / APM routes
├── group_vars/all.yml                  # topology, API, ports, timeouts
├── group_vars/vault.yml.example        # copy to vault.yml, then ansible-vault encrypt
├── playbooks/rolling_restart.yml
└── playbooks/tasks/
    ├── restart_es_node.yml
    └── restart_edge_node.yml
```

```bash
git clone https://github.com/nwlterry/elk_cluster_post_os_patching_restart.git
cd elk_cluster_post_os_patching_restart
```

## Before first run

1. Fill in `inventories/production.yml` (`ansible_host`, `es_node_name` must match `node.name`).
2. For APM, set `ansible_host` to the OpenShift Service or Route that listens on `apm_port`. Those hosts use `ansible_connection: local` and `skip_restart: true`.
3. Set `es_api_host` in `group_vars/all.yml` to a VIP or a master that stays reachable.
4. Store the password in Vault:

```bash
ansible-vault encrypt_string 'YOUR_PASSWORD' --name vault_es_api_password
```

5. Confirm units are enabled so they come back after reboot (not applicable to OpenShift APM):

```bash
systemctl is-enabled elasticsearch
systemctl is-enabled elastic-agent    # Fleet Server nodes
systemctl is-enabled logstash
systemctl is-enabled kibana
```

6. If Fleet is not using the `elastic-agent` unit name, set `fleet_service` in `group_vars/all.yml`. If APM listens on a different port than 8200, set `apm_port`.
7. Take a recent snapshot. Drop disk usage below the low watermark on any node that is close.

## Run

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml -e reboot_host=false --ask-vault-pass

# One tier
ansible-playbook playbooks/rolling_restart.yml --tags hot --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags fleet --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags apm --ask-vault-pass   # check only
ansible-playbook playbooks/rolling_restart.yml --tags logstash --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags kibana --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags edge --ask-vault-pass
```

`--tags precheck` still runs because precheck is tagged `always`. To skip it when targeting a single tier, add `--skip-tags always` only if the cluster is already green.

## What the playbook does not do

- It does **not** apply the OS patches. Run the patch playbook first, then this.
- It does **not** take snapshots.
- It does **not** use the node shutdown API (`PUT _nodes/{id}/shutdown`). That API is documented for ECE/ECK, not self-managed operators.
- It does **not** discover hosts from `_cat/nodes` or from the OpenShift API.
- It does **not** restart OpenShift APM pods. `skip_restart: true` is set on the `apm` group.
- Fleet / APM HTTP status checks treat `401` as “process is up” because those endpoints are often auth-gated.

## Tuning

| Variable | Default | Why |
|----------|---------|-----|
| `health_retries` × `health_poll_delay` | 180 × 15s = 45 min | Cold nodes on vSAN can take a long time to reallocate replicas |
| `reboot_timeout` | 900s | RHEL 8 + vSAN boot |
| `ml_set_upgrade_mode` | true | Only 2 ML nodes |
| `wait_for_no_relocating` | true | Do not start the next data node while shards are still moving |
| `allocation_disable_value` | `primaries` | Official setting |
| `fleet_port` / `apm_port` | 8220 / 8200 | Override if your policies differ |
| `skip_restart` | false (true on `apm`) | Skip reboot/systemd; HTTP/port check only |

## Recovery if the playbook stops mid-way

1. Check `_cluster/health` and `_cluster/settings`.
2. If `cluster.routing.allocation.enable` is still `primaries`, clear it:

```http
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

3. If ML jobs are stuck paused: `POST _ml/set_upgrade_mode?enabled=false`
4. Re-run with `--tags hot` / `--limit es-hot-04` or `--tags fleet --limit fleet-02` after that host is healthy.
