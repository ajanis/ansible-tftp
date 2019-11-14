# Ansible-TFTP

<!-- MarkdownTOC -->

- Requirements
- Role Variables
  - defaults/main.yml
  - vars/Debian|Ubuntu
  - vars/RedHat
- Dependencies
- Example Group Variables
- Example Playbook
- License
- Author Information

<!-- /MarkdownTOC -->

Set up a TFTP Server as part of the Foreman build server Project

This role is part of a project that will configure a Foreman build environment with optional TFTP and DHCP smart proxies, and an NGINX webserver for serving static content and acting as a reverse-proxy to Foreman.

## Requirements

N/A

## Role Variables
#### defaults/main.yml
```yaml
tftp_user: tftp
tftp_group: tftp

tftp_dir: /srv/tftp

tftp_pxe_dir:
  - boot
  - pxelinux.cfg

tftp_hpa_address: "0.0.0.0:69"
tftp_hpa_options: --secure

tftp_xinetd_socket_type: dgram
tftp_xinetd_protocol: udp
tftp_xinetd_wait: "yes"
tftp_xinetd_service_user: root
tftp_xinetd_server: /usr/sbin/in.tftpd
tftp_xinetd_server_args: "--user {{ tftp_user }} --secure {{ tftp_dir }}"
tftp_xinetd_disable: "no"
```
#### vars/Debian|Ubuntu
```yaml
tftp_pkg:
  - tftpd-hpa
  - syslinux
  - pxelinux

tftp_service: tftpd-hpa

tftp_syslinux_binary:
  - /usr/lib/PXELINUX/pxelinux.0
  - /usr/lib/PXELINUX/gpxelinux.0
  - /usr/lib/syslinux/modules/bios/menu.c32
  - /usr/lib/syslinux/modules/bios/chain.c32
  - /usr/lib/ipxe/undionly.kpxe
  - /usr/lib/ipxe/ipxe.efi
  - /usr/lib/ipxe/ipxe.lkrn
```
#### vars/RedHat
```yaml
tftp_pkg:
  - tftp-server
  - xinetd
  - syslinux

tftp_service: xinetd

tftp_syslinux_binary:
  - /usr/share/syslinux/pxelinux.0
  - /usr/share/syslinux/gpxelinux.0
  - /usr/share/syslinux/menu.c32
  - /usr/share/syslinux/chain.c32
  - /usr/share/ipxe/undionly.kpxe
  - /usr/share/ipxe/ipxe.efi
  - /usr/share/ipxe/ipxe.lkrn
```

## Dependencies

Additonal variables may be set in group_vars for configuring Foreman, nginx, isc-dhcp server, and tftp server


## Example Group Variables
```yaml
www_domain: home.example.com
foreman_hostname: foreman

fail2ban_enable: False

nginx_backends:
  - service: foreman
    servers:
      - localhost:3000

nginx_vhosts:
  - servername: "{{ foreman_hostname | default(ansible_hostname) + '.' + www_domain | default(ansible_domain) }}"
    serveralias: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }}"
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: foreman
      - name: 404.html
        docroot: "/usr/share/nginx/html"
      - name: 50x.html
        docroot: "/usr/share/nginx/html"
```

**PROTIP!!** If deploying isc-dhcp-server as part of foreman setup, use ```foreman_proxy_dhcp_subnets``` in your group_vars to configure both isc-dhcp-server and foreman DHCP smart-proxy.

```yaml
foreman_proxy_dhcp_subnets:
  - name: "10.0.10.0/24"
    network: "10.0.10.0"
    mask: "255.255.255.0"
    gateway: "10.0.10.1"
    from_ip: "10.0.10.240"
    to_ip: "10.0.10.250"
    vlanid:
    mtu: 9000
    domains:
    - "{{ www_domain }}"
    dns_primary: 10.0.10.1
    dns_secondary:
```

## Example Playbook
```yaml
- name: "Deploy Foreman Server"
  hosts: buildhost
  remote_user: root
  vars_files:
    - vault.yml
  tasks:

    - name: Wait for server to come online
      wait_for_connection:
        delay: 60
        sleep: 10
        connect_timeout: 5
        timeout: 900

    - include_role:
        name: common
      tags:
        - common

    - include_role:
        name: isc_dhcp_server
        public: yes
        apply:
          tags:
            - dhcp
      when: foreman_proxy_dhcp
      tags:
        - dhcp

    - include_role:
        name: tftp
        public: yes
        apply:
          tags:
            - tftp
      when: foreman_proxy_tftp
      tags:
        - tftp

    - include_role:
        name: nginx
        public: yes
        apply:
          tags:
            - nginx
      tags:
        - nginx

    - include_role:
        name: awx
        public: yes
        apply:
          tags:
            - awx
      tags:
        - awx

    - include_role:
        name: docker
        public: yes
        apply:
          tags:
            - docker
      tags:
        - docker

    - include_role:
        name: awx
        tasks_from: container-tasks.yml
        public: yes
        apply:
          tags:
            - awx
      tags:
        - awx

    - include_role:
        name: foreman
        public: yes
      tags:
        - install
        - configure
        - foreman
        - smartproxy
        - customize

    - include_role:
        name: ansible-project
        public: yes
      tags:
        - never
        - project
        - projectimport
        - projectclone
```

## License

MIT

## Author Information

Created by Alan Janis
