# devhub-ansible-proxmox

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Prepares an existing Proxmox VE installation for infrastructure automation: optional clustering, optional cloud-init VM templates and snippet storage, and a least-privilege, tool-neutral API identity.

The role is **node-agnostic**. Configure node-specific management addresses and, when building templates, storage, bridges and cluster-wide unique template VM IDs in inventory variables rather than editing the role.

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

## What it manages

- An opt-in Proxmox cluster: create it on the configured primary and print the manual join command for other nodes.
- Opt-in, checksum-verified cloud-init VM templates, using Ubuntu Minimal 24.04 LTS by default and supporting additional Linux cloud images through variables.
- Optional directory-backed snippet storage for custom cloud-init user-data or vendor-data, preserving existing storage content types.
- A dedicated `@pve` automation user, custom privilege role, API token and ACLs.
- An optional local `0600` environment file containing a newly created API token for an infrastructure-as-code client.

The role does not create or manage a Linux SSH account. The API token output uses the Proxmox provider's `PROXMOX_VE_*` environment variables for the API endpoint and token; TLS verification remains enabled. Trust the Proxmox certificate's issuing CA on the client and use an endpoint that matches the certificate.

## Requirements

Use Ansible 2.14 or newer with the `ansible.utils` collection and its `netaddr` dependency on the controller. Run the role with root privileges on a Proxmox VE node. Host networking and storage must already be configured; this simplified role does not manage repositories, bridges, host hardening, or create storage pools.

Template creation requires an active storage accepting `images` (by default `local-lvm`), an active Linux bridge (by default `vmbr0`), and HTTPS access from the node to the image and checksum URLs. Images must be x86-64 Linux cloud images with cloud-init already installed and compatible with the default SeaBIOS/virtio configuration.

## Cluster behavior

Clustering is disabled by default. A two-node cluster without a QDevice loses quorum when either node is offline; enable clustering only when its availability trade-offs are understood. Configure `proxmox_cluster_name`, `proxmox_cluster_primary` and, if needed, `proxmox_cluster_link_address`.

The primary creates the cluster. Other nodes are not joined automatically: Proxmox joining is interactive, so the role prints the `pvecm add` command for manual execution. Joining a node with existing VMs can erase their local VM configuration; the role refuses to proceed in that case.

## Cloud-init VM templates

Set `proxmox_build_templates: true` to enable template creation. The default image is **Ubuntu Minimal 24.04 LTS**, a smaller Ubuntu cloud image rather than an installer ISO or a container root filesystem. The role downloads and verifies the image, imports its disk, attaches a cloud-init CD-ROM, configures virtio networking and a serial console, and converts the stopped VM into a template. It never starts the base image, adds credentials, or installs packages inside it.

```yaml
# In the inventory variables for the node that will host the template:
proxmox_build_templates: true
proxmox_vm_storage: local-lvm
proxmox_bridge_name: vmbr0
```

The default template has VM ID `9000` and name `ubuntu-2404-minimal-cloudinit`. Clone it with your infrastructure-automation client, then set the clone's disk size, VLAN, SSH keys, user, and cloud-init networking before its first boot. The base NIC has no VLAN tag and defaults to DHCP; these are placeholders for clone-specific settings, not host network configuration.

### Additional images

Override `proxmox_vm_templates` with a list of images. Ansible replaces the default list rather than appending to it, so include Ubuntu Minimal in your list if you want to keep it:

```yaml
proxmox_vm_templates:
  - vmid: 9000
    name: ubuntu-2404-minimal-cloudinit
    url: https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
    checksum: sha256:https://cloud-images.ubuntu.com/minimal/releases/noble/release/SHA256SUMS
    cores: 2
    memory: 1024
  - vmid: 9001
    name: debian-13-cloudinit
    url: https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
    checksum: sha512:https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
    cores: 2
    memory: 1024
    bridge: vmbr0
    agent: false
```

Each entry requires `vmid`, `name`, `url`, and `checksum`. Optional fields are `cores` (default `2`), `memory` in MiB (default `1024`), `bridge` (default `proxmox_bridge_name`), and `agent` (default `false`). Checksums may be literal `sha256:<64 hex characters>` or `sha512:<128 hex characters>`, or HTTPS checksum-manifest URLs prefixed by their algorithm. HTTPS certificate validation remains enabled.

### Safety and image updates

VM IDs are unique across an entire Proxmox cluster, including containers. Build templates on one designated node, or assign different IDs to each node through host variables. If clustering is enabled, a node must have completed its cluster creation/join before building templates.

The role never overwrites or destroys existing guests. Existing templates are reused only when their managed marker matches the requested definition and their disk/cloud-init configuration is complete. Definition changes fail explicitly: use a new VM ID/name to roll out a new base without breaking existing clones. If conversion was interrupted after successful VM creation, a re-run completes the conversion; incomplete or unrelated VMs require manual inspection.

The default image URL follows Ubuntu's current release build. Existing templates do not refresh when that URL changes. Pin a dated image URL and a literal checksum for reproducible builds, and allocate a new VM ID/name for image updates. Verified downloads stay in `proxmox_image_cache_dir`, separated by image URL so identical filenames from different sources do not collide.

### Snippets and guest-agent discovery

With template building enabled, `proxmox_enable_snippets: true` enables the `snippets` content type on the existing `proxmox_snippets_storage` directory storage. The role creates its `snippets/` directory under the storage's configured path, not a hard-coded path. Existing `iso`, `backup`, and other content types are preserved. Set `proxmox_enable_snippets: false` if you only need Proxmox-generated cloud-init settings.

Enabling snippets does **not** create a Linux SSH user or implement snippet uploads. Your client needs a supported file-upload path and permissions; a Proxmox API token alone does not provide an SSH/SCP login. The role leaves the directory root-owned with mode `0755`.

Ubuntu Minimal is imported as-is. If your client relies on the QEMU guest agent to discover IP addresses, set `agent: true` on the template and install/start `qemu-guest-agent` in clones through cloud-init. For example, use the following as cloud-init vendor-data through your client's supported snippet mechanism, while retaining generated user-data for SSH keys:

```yaml
#cloud-config
packages:
  - qemu-guest-agent
runcmd:
  - [systemctl, enable, --now, qemu-guest-agent]
```

The `agent` option only enables the hypervisor's communication channel; it does not install a package. Clients waiting for IP discovery must allow time for the first-boot package installation and require working guest networking/package repositories.

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
| `proxmox_build_templates` | `false` | Opt into cloud image downloads and template creation |
| `proxmox_vm_templates` | Ubuntu Minimal 24.04 LTS, VM ID `9000` | List of cloud-init template definitions |
| `proxmox_vm_storage` | `local-lvm` | Existing active storage accepting VM images |
| `proxmox_bridge_name` | `vmbr0` | Existing active Linux bridge for template NICs |
| `proxmox_image_cache_dir` | `/var/lib/vz/template/cloud-images` | Node-side cache for verified images |
| `proxmox_enable_snippets` | `true` | Enable snippet storage when template building is enabled |
| `proxmox_snippets_storage` | `local` | Existing directory storage for custom cloud-init data |
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
  become: true
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

To select only template preparation after configuring the host:

```bash
ansible-playbook site.yml -i hosts -l <PVE_HOST> --tags templates \
  -e proxmox_build_templates=true
```

Template check mode validates inputs only; it does not download images, inspect live storage, or create VMs. A normal run is required to verify that storage, bridges, and completed templates are available.
