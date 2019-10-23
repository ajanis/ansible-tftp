Ansible-TFTP
=========

Set up a TFTP Server as part of the Foreman build server Project

This role is part of a project that will configure a Foreman build environment with optional TFTP and DHCP smart proxies, and an NGINX webserver for serving static content and acting as a reverse-proxy to Foreman.

Requirements
------------

N/A

Role Variables
--------------
```
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

Dependencies
------------

Additonal variables may be set in group_vars for configuring Foreman, nginx, isc-dhcp server, and tftp server


Example Group Variables
----------
```
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

isc_dhcp_server_subnet:
  - netaddress: 192.168.121.0
    netmask: 255.255.255.0
    gateway: 192.168.121.1
    domain: "{{ www_domain | default(ansible_domain) }}"
    domain_search: "{{ www_domain | default(ansible_domain) }}"
    dns: 192.168.121.1
    range: 192.168.121.20 192.168.121.100
```

Example Playbook
----------------
```
- name: "Deploy Foreman Server"
  hosts: foreman
  become: True
  remote_user: root
  tasks:


    - include_role:
        name: foreman
      tags:
        - install
        - configure
        - foreman
  
    - include_role:
        name: nginx
  
    - include_role:
        name: isc_dhcp_server
      when: foreman_proxy_dhcp

    - include_role:
        name: tftp
      when: foreman_proxy_tftp
```

License
-------

MIT

Author Information
------------------

Created by Alan Janis
