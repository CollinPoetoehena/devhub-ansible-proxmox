# devhub-ansible-proxmox

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures a machine that has been installed from the Proxmox VE ISO into a fully managed homelab hypervisor: free repositories, a VLAN-aware bridge on the tagged management VLAN (dual-stack), storage with cloud-init snippets, optional clustering, cloud-init VM templates, a least-privilege **automation account for Terraform**, host hardening, and verification.

The role is **node-agnostic**. Adding a second node (PVE2, the Mini PC) is an inventory entry plus a `host_vars/pve2.yml` with two variables — never an edit to this role.

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

---

## The one manual step: booting from the install medium

Everything in this repository is automated **except the initial install**, because a machine with no operating system cannot be reached over SSH. That is the entire manual scope. See for details on how to do this using a boot medium in [DevHub/reference/os_hardware/Booting_OS.md](https://github.com/CollinPoetoehena/DevHub/blob/main/reference/os_hardware/Booting_OS.md#hypervisor-installation-proxmox-ve)

Do this once per node, then never touch the physical machine again.

**Additional Step after installation: make the node temporarily reachable:** Log in as `root` on the console and give the node a temporary, correct, tagged address so Ansible can reach it (assuming the management VLAN is VLAN 10 and the node's temporary IP is 10.42.10.10; replace these values with your actual VLAN and IP if different):

```bash
# Confirm the NIC name — you need it for proxmox_physical_interface later.
ip -br link

# Temporary tagged management address on VLAN 10 (lost on reboot; the role makes it permanent). Replace eno1 with your NIC.
ip link add link eno1 name eno1.10 type vlan id 10
ip addr add 10.42.10.10/24 dev eno1.10
ip link set eno1 up && ip link set eno1.10 up
ip route add default via 10.42.10.1

# Verify you can reach the router and the internet through the proxmox node.
ping -c3 10.42.10.1
ping -c3 1.1.1.1

# Install your lab key for root (run this FROM your laptop, not on the node), such as:
ssh-copy-id -i ~/.ssh/id_homelab.pub root@10.42.10.10

# On the node: a laptop lid must not suspend a hypervisor.
sed -i 's/^#\?HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sed -i 's/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
systemctl restart systemd-logind
```

**That is the end of the manual work.** From here the node is headless and everything below is automated.

---

## What this role does

| Task file | Purpose |
|---|---|
| `preflight.yml` | Refuse to run against a non-Proxmox host; assert per-node variables; carrier check |
| `repositories.yml` | Disable the enterprise repo, enable no-subscription, dist-upgrade, remove the subscription nag |
| `packages.yml` | Utility + dependency packages; `ifupdown2` for live network reloads |
| `networking.yml` | VLAN-aware `vmbr0`, tagged management interface, dual-stack addressing, `accept_ra=2` |
| `storage.yml` | Enable the `snippets` content type, create the image cache, validate storages |
| `cluster.yml` | Create/join a Proxmox cluster (off by default — see below) |
| `templates.yml` + `template_build.yml` | Build cloud-init golden images that Terraform clones |
| `automation.yml` | **The Terraform account**: PVE role, group, user, API token, ACL, SSH system user |
| `hardening.yml` | SSH hardening, Proxmox cluster/host firewall |
| `verify.yml` | Read-only checks + a single printed summary |

---

## Networking

The switch delivers a trunk to this node. The node therefore puts the physical NIC into a **VLAN-aware bridge** and gives *itself* an address only on the tagged management VLAN:

```
lab switch port 2 (trunk: untagged 1 + tagged 10/20/30)
      │
  eno1            physical NIC — no IP, bridge member only
      │
  vmbr0           VLAN-AWARE bridge — VM NICs attach here and carry their own tag
      │
  vmbr0.10        the NODE's own management address (10.42.10.10/24 + SLAAC)
```

**Why VLAN-aware and not one bridge per VLAN.** A VLAN-aware bridge passes 802.1Q tags through to VM NICs instead of stripping them, so placing a VM on VLAN 20 is a one-line tag on that VM — exactly what the Terraform module's `vlan_id` variable sets. The alternative (vmbr0.10, vmbr0.20, vmbr0.30 as separate bridges) pins each VM to one bridge and makes moving it between VLANs a reconfiguration of its virtual hardware.

**Why the node has no address on `vmbr0` itself.** That would be the untagged native VLAN, which carries no production traffic in a typical lab by design (such as in my personal homelab in the [DevHub repository/homelab](https://github.com/CollinPoetoehena/DevHub/tree/main/homelab)) — see [Why VLAN 1 is not used](https://github.com/CollinPoetoehena/DevHub/tree/main/homelab/docs/1_Design/2_Design_Network.md#why-vlan-1-native-is-not-used--all-traffic-is-explicitly-vlan-tagged).

**Why IPv4 is static but IPv6 is SLAAC.** The API endpoint, certificate SANs, corosync's link address and every firewall rule referencing this node want a stable v4 address. IPv6 comes from the router's Router Advertisements, and the router's `ra-names` already publishes the matching AAAA record — so the node is reachable by name over IPv6 with zero extra configuration, which is the habit the lab is deliberately building.

**`accept_ra` must be 2.** A host that forwards packets — which any hypervisor with a bridge does — ignores Router Advertisements by default (`accept_ra=1` means "accept only if forwarding is off"). The node then gets an IPv6 address from the prefix but **no default route**, and "IPv6 works on the LAN but not to the internet" is the confusing result. This is the same trap the router role documents for its WAN interface.

**Safety.** `/etc/network/interfaces` is rendered to a staging file, validated with `ifreload --syntax-check`, backed up, then installed and applied with `ifreload -a` (a diff-based reload, not a full `down`/`up`). If the syntax check fails, the live file is never touched. This is the network equivalent of the router role's `nft --check`.

---

## The automation account

This is the handover point between "Ansible owns the host" and "Terraform owns the VMs on it". **Three** things are created, and they are genuinely different:

| What | Name | Purpose |
|---|---|---|
| PVE role | `TerraformProv` | A named bundle of privileges |
| PVE user + API token | `terraform@pve!automation` | The identity Terraform authenticates as over the REST API |
| Linux system user | `terraform` | The SSH transport, for the few operations the API does not expose |

**Why not just use `root@pam`:**

1. **Least privilege** — creating a VM does not require the ability to reinstall the node, rewrite its network or add users. The role grants VM/Datastore/Pool privileges and deliberately omits `Sys.Modify`, `Realm.Allocate`, `Sys.PowerMgmt` and `Permissions.Modify`.
2. **Revocable** — a token is deleted in one click without touching any human account.
3. **Auditable** — every Proxmox task log entry carries the token that performed it.
4. **Non-interactive** — a token has no password, no TTY and no 2FA prompt.

**Why realm `@pve` and not `@pam`:** a `@pam` user is a real Linux account that would have to exist, with matching keys, on every node. A `@pve` user lives in `/etc/pve` — the replicated cluster filesystem — so it is created once and appears automatically on every node that later joins.

**Why the ACL is on `/` with propagate:** Terraform allocates *new* VM ids, which have no ACL path of their own yet, and it reads node and storage status before creating anything. Scoping to `/vms` breaks `terraform plan` against empty state. The blast radius is limited by the **role** (what may be done), not by the path.

**Why the token also needs its own ACL entry:** with `privsep=1` a token starts with *zero* permissions even though its user has them all. Forgetting that one `pveum acl modify --tokens` call is the single most common reason a brand-new token returns `403 Permission check failed` for every call. `verify.yml > "VERIFY AUTOMATION"` checks for exactly this.

**Why a separate SSH account:** the `bpg/proxmox` provider does almost everything over the API, but uploading a cloud-init snippet and importing a disk image have no API equivalent and are done over SSH. That account gets key-only login and passwordless sudo restricted to `qm`, `pvesm` and snippet writes — blanket `NOPASSWD:ALL` would hand back through a different door exactly the privileges the API role withholds.

### The token secret

Proxmox displays a token secret **exactly once**, at creation. The role captures it and writes it to the controller as a `0600` env file:

```bash
source ansible/.secrets/pve1-terraform-token.env   # add .secrets/ to .gitignore!
cd terraform/environments/homelab && terraform plan
```

**Lost it? Rotate:**

```bash
ansible-playbook site.yml -l pve1 --tags automation -e proxmox_rotate_automation_token=true
```

---

## VM templates

Terraform **clones**, it never installs an OS. Ansible prepares the golden image once:

download cloud image → verify checksum → `virt-customize --install qemu-guest-agent` → `qm create` → `qm importdisk` → attach cloud-init drive → `qm template`.

- **Why the guest agent is injected into the image**: without it the hypervisor cannot read a VM's IP (the Terraform module's `ipv4_addresses` output stays empty forever), cannot shut down gracefully, and cannot quiesce the filesystem for snapshots. It must be in the *image*, because a VM that has not booted yet cannot install it.
- **Why `--truncate /etc/machine-id`**: systemd derives the DHCP client identifier from the machine-id. If every clone inherits the same one, every VM requests the **same lease** from the router's dnsmasq and they fight over one address.
- **Why `qm template`**: it makes the image read-only and enables *linked* clones, which store only their differences — a 20 GB template plus ten VMs costs far less than 200 GB.

---

## Clustering (off by default — deliberately)

Corosync requires **quorum**: a strict majority of votes. In a two-node cluster the majority is 2, so if either node is off the survivor has 1 of 2, loses quorum, and `/etc/pve` goes **read-only** — no VM can be started, created or modified on an otherwise healthy node. In a homelab where the second machine is often powered off, that is strictly worse than two standalone nodes.

- PVE1 alone → `proxmox_cluster_enabled: false` (correct today)
- PVE1 + PVE2 always on → enable it
- PVE1 + PVE2 sometimes → enable it **and** add a QDevice for a third vote (the always-on lab router is the obvious host)

> **Joining a cluster wipes the VMs already defined on the joining node.** Cluster first, create VMs second.

---

## Requirements

- A machine installed from the Proxmox VE ISO (see the manual step above), PVE 8 or 9
- Its NIC connected to a trunk port on the lab switch (tagged 10/20/30)
- Key-based SSH access as `root` (or a user with sudo)
- The lab router configured and running (this role assumes DHCP/DNS/RA and the firewall already work)
- Ansible collections: `community.general`, `ansible.posix`, `ansible.utils`

---

## Key variables

### Required per node (`host_vars/pve1.yml`)

| Variable | Description |
|---|---|
| `proxmox_mgmt_ipv4` | Management address, e.g. `10.42.10.10` |
| `proxmox_physical_interface` | NIC facing the lab switch, e.g. `eno1` (`ip -br link`) |

### Commonly set (`group_vars/proxmox/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `proxmox_domain` | `lab.local` | Must match `router_dhcp_domain` |
| `proxmox_mgmt_gateway` | `10.42.10.1` | Router's VLAN 10 address |
| `proxmox_mgmt_vlan_id` | `10` | Management VLAN |
| `proxmox_bridge_name` | `vmbr0` | VM-facing bridge |
| `proxmox_bridge_vlan_aware` | `true` | **Required** for per-VM VLAN tags |
| `proxmox_bridge_vlan_ids` | `2-4094` | VLANs the bridge accepts |
| `proxmox_enable_ipv6` | `true` | Dual-stack |
| `proxmox_mgmt_ipv6_method` | `auto` | `auto` (SLAAC) / `static` / `disabled` |
| `proxmox_vm_storage` | `local-lvm` | Where VM disks live |
| `proxmox_snippets_storage` | `local` | Must be a *directory* storage |
| `proxmox_automation_ssh_keys` | `[]` | **Set this** — otherwise Terraform's SSH operations fail |
| `proxmox_vm_templates` | Debian 13 + Ubuntu 24.04 | Templates to build |
| `proxmox_cluster_enabled` | `false` | See the clustering section |
| `proxmox_firewall_management_sources` | `[10.42.10.0/24]` | Who may reach SSH/8006 |

Full schema and rationale for every variable: [`defaults/main.yml`](defaults/main.yml).

---

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

Run the playbook:
```bash
cd homelab; source venv/bin/activate; cd ansible
eval $(ssh-agent) && ssh-add ~/.ssh/id_homelab

# Dry run first — --diff shows the exact file changes, -C changes nothing.
ansible-playbook site.yml -i hosts -l pve1 --tags proxmox --diff -C

# Apply
ansible-playbook site.yml -i hosts -l pve1 --tags proxmox --diff

# Individual concerns (each tag is self-contained)
ansible-playbook site.yml -i hosts -l pve1 --tags network --diff
ansible-playbook site.yml -i hosts -l pve1 --tags templates
ansible-playbook site.yml -i hosts -l pve1 --tags automation
ansible-playbook site.yml -i hosts -l pve1 --tags verify
```

> The first run connects as `root` (or your install user). Once `automation.yml` has run, normal runs can use the ansibleremote pattern used elsewhere in this repo.

### Handy manual checks

```bash
ip -br addr show vmbr0.10                 # management addresses, both families
cat /sys/class/net/vmbr0/bridge/vlan_filtering   # must be 1
bridge vlan show dev eno1                 # which VLANs the trunk carries
ip -6 route show default                  # empty => accept_ra problem
rdisc6 vmbr0.10                           # inspect the router's RA
pvesm status                              # storages and content types
qm list                                   # VMs and templates (9000+)
pveum user permissions terraform@pve --token automation
pve-firewall status && tail -f /var/log/pve-firewall.log
```

### Accessing the web UI

The management VLAN is not routed from your home laptop by design. Tunnel through the lab router, exactly as with the switch (change `<user>` and `<lab-router-ip>` to the actual values of your lab router and the `-i` option to the correct SSH key):

```bash
ssh -L 8006:10.42.10.10:8006 <user>@<lab-router-ip> -i ~/.ssh/id_homelab
# then browse to https://localhost:8006
```

---

## Tags

- `proxmox` — everything
- `repositories`, `apt` — repository configuration and upgrades
- `packages` — utility/dependency packages
- `network`, `networking` — bridge, VLAN interface, addressing, sysctls
- `storage` — content types, snippets, image cache
- `cluster` — cluster create/join
- `templates` — build cloud-init golden images (downloads ~500 MB)
- `automation`, `terraform` — the Terraform role/user/token/ACL/SSH account
- `hardening`, `security` — SSH and the Proxmox firewall
- `verify` — read-only checks and the summary
