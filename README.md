# Ansible Roles: Proxmox

This is a role to setup a new container.  Note that you must have root ssh to a proxmox node configured.  You must also have api access to your proxmox host/cluster.

This role will primarily run on the ansible localhost.  Some tasks will use the "delegate_to" option to run on the remote Proxmox node.

## Role Variables

Lots of variables are available to use for provisioning a container.  Required will depend on your use case.  Most are optional.

Simple Container:

```yaml
pve_node: pve1
pve_apiuser: root@pam
pve_apipass: myAPIpassword
pve_hostname: "newhostname"
pve_template: local:vztmpl/debian-9.0-standard_9.5-1_amd64.tar.gz
```

Detailed example with bind mounts from the Proxmox node.  I added a cephfs mount to my cluster, so it's mounted on each Proxmox node:

```yaml
pve_node: pve1
pve_vmid: 114
pve_apiuser: engonzal@pve
pve_apipass: myAPIpassword
pve_api_host: pve1.domain.com
pve_hostname: "newhostname"
pve_template: local:vztmpl/debian-9.0-standard_9.5-1_amd64.tar.gz
pve_netif:
  net0: "name=eth0,gw=192.168.84.1,ip=192.168.84.20/24,bridge=vmbr0"
pve_cores: 2
pve_mem: 2048
pve_swap: "{{ pve_mem }}"
pve_guest_pass: myContainerRootPassword
pve_search: domain.com
pve_dns: '192.168.84.1'
pve_storage: ceph_storage_ct
pve_unprivileged: yes
pve_ssh: "ssh-rsa myPublicKey engonzal@hostname"
pve_custom_mounts:
  mp0: "/mnt/pve/cephfs_data/downloads/,mp=/downloads"
  mp1: "/mnt/pve/cephfs_data/media,mp=/media"
```

## Example Playbooka

Ansible hosts inventory file

```yaml
# hosts
[proxmox_containers]
test_server
```

Ansible playbook

```yaml
# proxmox.yml
---
- hosts: plex_app
  connection: local
  user: root
  vars:
    pve_node: pve1
    pve_apiuser: root@pam
    pve_apipass: myAPIpassword
    pve_api_host: pve1.domain.com
    pve_hostname: "newhostname"
    pve_template: local:vztmpl/debian-9.0-standard_9.5-1_amd64.tar.gz
  roles:
    - engonzal.proxmoxct
```

Ansible run command

```bash
ansible-playbook -i hosts -l test_server proxmox.yml
```

### Example Playbook (advanced)

You can also add a delay after your play if you have other plays to run after:

```yaml
---
- hosts: plex_app
  connection: local
  user: root
  pre_tasks:
  - name: get current python interpreter (for pip virtualenvs)
    command: which python
    register: which_interpreter
    tags: always
    changed_when: False

  - name: Use the current python path instead of system python
    set_fact:
      ansible_python_interpreter: "{{ which_interpreter.stdout }}"
    tags: always

  roles:
    - name: engonzal.proxmoxct
      tags: pve
  post_tasks:
    - name: Allow container time to boot if started
      pause:
        seconds: 20
      when: pve_info_state.changed

- hosts: plex_app
  user: root
  vars:
    ansible_python_interpreter: /usr/bin/python3 # needed for bionic ct
    package_list:
      - vim
  roles:
    - engonzal.package
```

## Other Examples

### DHCP Example:

```yaml
pve_node: pve1
pve_vmid: 114
pve_apiuser: engonzal@pve
pve_apipass: myAPIpassword
pve_api_host: pve1.domain.com
pve_state: present
pve_hostname: "newhostname"
pve_template: local:vztmpl/debian-9.0-standard_9.5-1_amd64.tar.gz
pve_netif:
  net0: "name=eth0,ip=dhcp,ip6=dhcp,bridge=vmbr0"
pve_storage: local-lvm
pve_custom_mounts:
  mp0: "/mnt/pve/cephfs_data/downloads/,mp=/downloads"
  mp1: "/mnt/pve/cephfs_data/media,mp=/media"
```

### Organization

For my uses I organize the variables like this (using ansible vault to encrypt passwords):

```yaml
# group_vars/all
pve_apiuser: engonzal@pve
pve_apipass: myAPIpassword
pve_api_host: pve1.domain.com
pve_guest_pass: myContainerRootPassword
pve_search: domain.com
pve_dns: '192.168.84.1'
pve_unprivileged: yes
pve_ssh: "ssh-rsa myPublicKey engonzal@hostname"

# group_vars/plex
pve_node: pve3
pve_vmid: 114
pve_hostname: "plex"
pve_netif:
  net0: "name=eth0,gw=192.168.84.1,ip=192.168.84.20/24,bridge=vmbr0"
pve_template: local:vztmpl/ubuntu-18.10-standard_18.10-1_amd64.tar.gz
pve_cores: 8
pve_mem: 4096
pve_custom_mounts:
  mp0: "/mnt/pve/cephfs_data/media,mp=/media"
```

#### License

BSD

## Notes
Proxmox api info available at:  

* <https://pve.proxmox.com/pve-docs/api-viewer/index.html>
* <https://pve.proxmox.com/wiki/Proxmox_VE_API>
