# elk_cluster_post_os_patching_restart

Ansible rolling **host reboot** after monthly OS patching. Node order follows Elastic rolling restart (same version; no package upgrade in this playbook).

## Layout

```
playbooks/rolling_restart.yml
playbooks/tasks/          # ES node, edge node, mute/unmute
inventories/production.yml
group_vars/all.yml
group_vars/vault.yml.example
docs/AZURE_DEVOPS_WIKI.md
docs/objects/             # cluster-status rule JSON copies
ansible.cfg
GROUP.md
README.md
```

Rules source: [elk_stack_monitor](https://github.com/nwlterry/elk_stack_monitor).

```bash
ansible-playbook playbooks/rolling_restart.yml --tags precheck --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --ask-vault-pass
ansible-playbook playbooks/rolling_restart.yml --tags unmute --ask-vault-pass
```

Indexing stays on. **No `POST /_flush`.** OpenShift APM is check-only.

---

See [GROUP.md](GROUP.md) for sibling repositories. Catalog: https://github.com/nwlterry/nwlterry
