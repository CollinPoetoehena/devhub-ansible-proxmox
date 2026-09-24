# devhub-ansible-proxmox

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures a Proxmox host with Ansible for lab networking and management.

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

**TODO: fill this in below further when this role is fully done:**
## Requirements

- Proxmox VE installed on the host machine via a boot device: see [DevHub/reference/os_hardware/Booting_OS.md](https://github.com/CollinPoetoehena/DevHub/blob/main/reference/os_hardware/Booting_OS.md#hypervisor-installation-proxmox-ve)
- A network connection for the control machine to communicate with the Proxmox host and SSH access.
- Python and Ansible installed on the control machine to run the playbooks.
- TODO: any others??

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| | | |

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
