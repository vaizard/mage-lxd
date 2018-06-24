# mage-lxd

Ansible role to install and configure the lxd service and manage (local and remote) lxd containers.

CAVEATS:

- To keep up with the most recent lxd releases, this role is intentionally hardcoded to use snaps (instead of the packaged version). This is ok for us, becuse we use Ubuntu exclusively for LXD hosts. If anyone feels like adding support for system-packaged lxd, just commit a PR.
- This role is hardcoded for clustering (can be easily disabled) and thus uses Ubuntu's fan networking (which is currently hardcoded). Using bridged networking should be an easy fix, if you'd like to send a PR.
- In order to use ansible's lxd connector, you must have lxd installed on the machine you run ansible on. For that reason, the lxd-local.yml installs the lxd snap. If you don't want to use snap, just skip the local tag and install lxd manually.
- This role has been tested exclusively on Ubuntu 18.04. Newer releases should work just fine, older probably won't work at all.
- This role must have the `{{ inventory_file }}` variable defined, as the containers-outer.yml task adds created lxd containers to the inventory file. As recent Ansible doesn't have this variable defined at runtime, you must set it manually here.

```
---
- hosts: lxdhost01
  vars:
    inventory_file: /home/user/ansible/linux/inventories/prod/hosts
    lxd_snap_channel: edge
    lxd_snap_refresh: true
    lxd_init_run: false
    lxd_init_https_addr: 10.1.2.1
    lxd_init_https_port: 8443
    lxd_init_trust_pass: my-super-secret-trustpass
    lxd_init_clust_name: "{{ ansible_hostname }}"
    lxd_container_server: "https://images.linuxcontainers.org"
    lxd_container_protocol: "simplestreams"
    lxd_container_alias: "ubuntu/bionic/amd64"

    lxd_containers:
    - name: "c1"
      devices:
        "http_alt": { "listen": "tcp:10.1.2.1:8080", "connect": "tcp:127.0.0.1:8080", "bind":"host", "type": "proxy" }
        "ftprange": { "listen": "udp:10.1.2.1:9000-9200", "connect": "udp:127.0.0.1:9000-9200", "bind":"host", "type": "proxy" }
      proxy: 
        - { name: "https", dst: 443, nxt: 443, proto: tcp }
        - { name: "ftpsrange", dst: 12000-12500, nxt: 12000-12500, proto: udp }
    - name: "c2"
      image: "fedora/27"
      proxy: 
        - { name: "https", dst: 443, nxt: 443, proto: tcp }
        - { name: "ftpsrange", dst: 12000-12500, nxt: 12000-12500, proto: udp }
  roles:
    - { role: mage-lxd, tags: [ 'install', 'configure', 'local', 'manage' ] }
```

TIPS:

- To get the role's tags run `ansible-playbook lxdhost01.yml --list-tags`.
- For other things that can be configured on the lxd containers, please refer to the containers-inner.yml task.
- The `lxd_containers.item.devices` can be used to configure **any** lxd device. Since the most common device
  we use is the proxy device, we also add `lxd_containers.item.proxy`, than allows a well readable configuration,
  that can also be used by our mikrotik/firewall roles to properly configure portforwarding.
