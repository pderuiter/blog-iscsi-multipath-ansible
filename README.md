# blog-iscsi-multipath-ansible

Companion code for the blog post
[Automating iSCSI Multipathing with Ansible](https://blog.bsdserver.nl/automating-iscsi-multipathing-with-ansible/).

It builds both ends of the setup from the post:

- **storage01**: a Linux LIO iSCSI target (`targetcli-fb`) with two portals, one on each storage
  network, that exports its second disk as LUN 0 with one-way CHAP.
- **node01**: an initiator configured by the `iscsi_multipath` role from the post: open-iscsi with
  CHAP, a login to both portals, automatic login at boot and dm-multipath (`mpatha`, multibus,
  round-robin) over the two paths.

Terraform creates the two Ubuntu 24.04 VMs on Proxmox VE, each with one NIC on each of two bridges;
Ansible configures them.

## Requirements

- Terraform >= 1.9 and a Proxmox VE API token (the bpg/proxmox provider, pinned in `terraform/versions.tf`).
  Two bridges (one per storage network); the datastore for the cloud image needs the `import` content type.
- Ansible (ansible-core 2.17 or later; tested with 2.22). Only `ansible.builtin` modules are used.
- Ubuntu 24.04 on the target and the initiators.

## Quick start

```bash
# 1. VMs
cd terraform
cp terraform.tfvars.example terraform.tfvars   # endpoint, token, node, bridges, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Secrets
cd ../ansible
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # set the CHAP username and password
ansible-vault encrypt group_vars/all/vault.yml

# 3. Configure target and initiator, then run the checks from the post
ansible-playbook site.yml --ask-vault-pass
ansible-playbook verify.yml --ask-vault-pass
```

No Proxmox? Skip step 1 and copy `ansible/inventory.example.yml` to `ansible/inventory.yml`. The target
needs an unused disk for the LUN and an address on each storage network.

## How it fits together

`site.yml` runs three plays:

1. read the IQN from `/etc/iscsi/initiatorname.iscsi` on every initiator (the post leaves the
   initiator name alone, so the target ACL uses the existing one);
2. configure the target (`iscsi_target`): `/etc/rtslib-fb-target/saveconfig.json`, restored by the
   `rtslib-fb-targetctl` service;
3. the play from the post (`iscsi_multipath`), limited to the `iscsi_initiators` group.

The post retrieves the WWID with `multipath -ll` and pastes it into `vars/main.yml`. Here the target
gives the LUN a fixed serial (`iscsi_target_luns[0].wwn`), so `vars/main.yml` computes the WWID:
`36001405` followed by the first 25 hex digits of the `wwn` without dashes. If you use your own
target, replace it with the WWID from `multipath -ll`.

## Variables

Set in `ansible/group_vars/all/vars.yml`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `iscsi_target_ips` | `10.0.10.11`, `10.0.20.11` | Both target portals (from the post) |
| `iscsi_target_name` | `iqn.2025-03.com.example:storage` | Target IQN (from the post) |
| `iscsi_chap_username` / `iscsi_chap_password` | from vault | CHAP credentials (`vault_iscsi_chap_*` in `vault.yml`) |
| `iscsi_target_luns` | `/dev/sdb` as `lun0` | Block devices exported by the target, with a fixed `wwn` (UUID) |
| `iscsi_target_chap_enabled` | `true` | One-way CHAP on every ACL |

Note: the target host must not use the LUN disk itself. If you put LVM on `mpatha`, the target sees
those PVs on its backing disk as well; keep it out of them (for example with an LVM `global_filter`).

## Tested

Lab-tested on Ubuntu 24.04 (kernel 6.8) with ansible-core 2.22 and two portals: the first run configures
both ends, the second run reports `changed=0`, and `verify.yml` passes (one session per portal, `mpatha`
with two `active ready running` paths). Taking one storage path down during writes gave no I/O errors
(one write paused for about 15 seconds), and the path came back by itself. Terraform is checked with
`fmt -check`, `validate` and `tflint`; `terraform apply` was not run as part of these tests.

## License

MIT
