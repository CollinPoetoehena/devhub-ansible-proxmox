# devhub-ansible-proxmox

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures a machine that has been installed from the Proxmox VE ISO into a fully managed homelab hypervisor: free repositories, a VLAN-aware bridge on the tagged management VLAN (dual-stack), storage with cloud-init snippets, optional clustering, cloud-init VM templates, a least-privilege **automation account for Terraform**, host hardening, and verification.

The role is **node-agnostic**. Adding a second node (PVE2, the Mini PC) is an inventory entry plus a `host_vars/pve2.yml` with two variables — never an edit to this role.

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

## What it manages

- An opt-in Proxmox cluster: create it on the configured primary and print the manual join command for other nodes.
- A dedicated `@pve` automation user, custom privilege role, API token and ACLs.
- An optional local `0600` environment file containing a newly created API token for an infrastructure-as-code client.

The role does not create or manage a Linux SSH account. The API token output uses the Proxmox provider's `PROXMOX_VE_*` environment variables for the API endpoint and token; TLS verification remains enabled. Trust the Proxmox certificate's issuing CA on the client and use an endpoint that matches the certificate.

## Cluster behavior

Clustering is disabled by default. A two-node cluster without a QDevice loses quorum when either node is offline; enable clustering only when its availability trade-offs are understood. Configure `proxmox_cluster_name`, `proxmox_cluster_primary` and, if needed, `proxmox_cluster_link_address`.

The primary creates the cluster. Other nodes are not joined automatically: Proxmox joining is interactive, so the role prints the `pvecm add` command for manual execution. Joining a node with existing VMs can erase their local VM configuration; the role refuses to proceed in that case.

## API identity and token

The role creates an `automation@pve` user without a password and assigns a custom role containing the configured infrastructure-provisioning privileges. It attaches the user to an automation group and grants the role to that group; with privilege separation enabled, it also grants the role directly to the token. The default privilege list supports VM provisioning while avoiding hypervisor administration and permission-management privileges.

### Why the identity is tool-neutral

The Proxmox user, group, role and token represent an infrastructure-automation identity and its permissions, not a particular client. Naming them after a client such as Terraform would make a future switch to another tool appear to require renaming Proxmox objects, changing ACLs and replacing credentials even when the required capabilities have not changed. A stable, purpose-based identity avoids that coupling and keeps access policy focused on what the automation may do. Client-specific integration details, such as the `PROXMOX_VE_*` environment-variable names used by the current provider, stay at the handoff boundary rather than defining the identity.

Proxmox displays an API token secret only once. On token creation, the role can save it on the Ansible controller in a `0700` directory and a `0600` file:

```bash
source ansible/.secrets/<PVE_HOST>-proxmox-api-token.env
# Run the desired infrastructure-as-code client.
```

The directory is ignored by Git, but the file is still a credential: do not copy it into logs or share it. The role does not rewrite it when the token already exists because Proxmox cannot return the secret a second time. To rotate a lost or compromised token:

```bash
ansible-playbook site.yml -l <PVE_HOST> --tags automation \
  -e proxmox_rotate_automation_token=true
```

## Variables

Set `proxmox_mgmt_ipv4` for each node in `host_vars`; it is used to build the API endpoint and as the default cluster link address.

| Variable | Default | Purpose |
| --- | --- | --- |
| `proxmox_cluster_enabled` | `false` | Enable cluster creation/status handling |
| `proxmox_cluster_name` | `homelab` | Name used when creating the cluster |
| `proxmox_cluster_primary` | First host in `proxmox` group | Node that creates the cluster |
| `proxmox_cluster_link_address` | `proxmox_mgmt_ipv4` | Corosync link address |
| `proxmox_automation_user` | `automation@pve` | Infrastructure automation API user |
| `proxmox_automation_token_id` | `automation` | API token identifier |
| `proxmox_automation_role` | `InfrastructureProvisioner` | Custom Proxmox role |
| `proxmox_automation_group` | `automation` | Group granted the custom role |
| `proxmox_automation_privileges` | See [`defaults/main.yml`](defaults/main.yml) | Privileges assigned to the role |
| `proxmox_automation_acl_path` | `/` | ACL path for the role |
| `proxmox_automation_acl_propagate` | `true` | Propagate the ACL to child paths |
| `proxmox_automation_token_privsep` | `true` | Give the token its own ACL |
| `proxmox_api_endpoint` | HTTPS endpoint from `proxmox_mgmt_ipv4` | Endpoint written to the token env file |
| `proxmox_automation_token_output` | `.secrets/<inventory-host>-proxmox-api-token.env` | Controller-side secret file |
| `proxmox_write_automation_token_file` | `true` | Save a newly created token to that file |
| `proxmox_rotate_automation_token` | `false` | Remove and recreate the token |

## Usage

Requirements file example (same directory as ansible.cfg, create a file called requirements.yml):

```yaml
---
roles:
  - name: devhub.proxmox
    src: https://github.com/CollinPoetoehena/devhub-ansible-proxmox.git
    scm: git
    version: 1.0.0
```

Then install with:

```sh
# NOTE: Example of roles path for -p is "roles/" (you can also specify this in ansible.cfg)
ansible-galaxy install -r requirements.yml -p <path/to/roles>
```

Example playbook using this role (e.g. site.yml):
```yaml
- hosts: all
  roles:
    - role: devhub.proxmox
```

Validate the playbook before applying it:

```bash
ansible-playbook site.yml -i hosts --syntax-check
ansible-playbook site.yml -i hosts -l <PVE_HOST> --check
ansible-playbook site.yml -i hosts -l <PVE_HOST>
```

`--check` skips or cannot reliably predict the Proxmox CLI commands. Review the effects before running against a real host.
