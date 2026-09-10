# elk_cluster_post_os_patching_restart

Ansible rolling restart for a self-managed Elastic Stack after monthly OS patching.

https://github.com/nwlterry/elk_cluster_post_os_patching_restart

Automates the official Elasticsearch rolling-restart procedure, then rolls Fleet Server, Logstash, and Kibana one host at a time. Both APM servers run as OpenShift containers and are **health-checked only** (no reboot, no systemd).

ES data-node loop:

1. Cluster must be **green**
2. On each **data** node: `cluster.routing.allocation.enable = primaries`
3. Reboot the host (or restart the service)
4. Wait until the node rejoins `_cat/nodes`
5. Clear `allocation.enable` (back to default)
6. Wait for **green** and no relocating / initializing shards
7. Next node

Edge nodes wait for ES green, reboot/restart their own systemd unit, then wait for port + optional HTTP status. Hosts with `skip_restart: true` skip reboot and systemd and only run the port/HTTP check (APM on OpenShift).

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

## Node order

Elastic dedicated-master-first start sequence, then ingest / UI that depend on a green cluster:

| Step | Group | Count | Allocation disable | Notes |
|------|-------|-------|--------------------|-------|
| 1 | `es_masters` | 3 | no | Quorum stays 2/3 |
| 2 | `es_data_hot` | 6 | yes | Wait green after every node |
| 3 | `es_data_cold` | 5 | yes | Same |
| 4 | `es_ml` | 2 | no | Jobs paused via `_ml/set_upgrade_mode` |
| 5 | `fleet` | 2 | n/a | After ES green; agents enroll here |
| 6 | `apm` | 2 | n/a | OpenShift containers — port/HTTP check only |
| 7 | `logstash` | 2 | n/a | Writes to ES; keep one up |
| 8 | `kibana` | 2 | n/a | UI last |

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
- It does **not** restart OpenShift APM pods. Set `skip_restart: true` (already set on the `apm` group).
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
