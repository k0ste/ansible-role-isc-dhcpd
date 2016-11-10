dhcp_server
===============

This role installs and configures a DHCP server.

Ansible versions
--------------------
Role is adapted for Ansible 2.0, should work on 1.9.

Requirements for usage
-----------------------------------
* [python-netaddr](//docs.ansible.com/ansible/playbooks_filters_ipaddr.html)
library (on machine with Ansible).

Ubuntu AppArmor
------------------
Since Ubuntu 14.04, AppArmor is configured to not allow dhcpd to access files
outside a certain list of paths.
This prevents Ansible from running the check command on the template. The check
is used to validate the correctness of the config file generated.

To prevent this, you can either disable AppArmor, manually configure it in such
a way that it allows access to `/root/.ansible/tmp` for dhcpd or you can let
this role do that for you:

If you specify the `dhcpd_configure_apparmor: true` variable for your host. This
role will overwrite the `/etc/apparmor.d/local/usr.bin.dhcpd` file and
specifically allow read-only access to `/root/.ansible/tmp`. It will first check
if this file exists, if it does not, it will not do anything.

Package Installation
---------------------------
If you don't need install package, just deploy (save your time), disable package
installation by set variable `dhcpd_install_package: false`.

Handlers
------------
You can control handlers behavior via variables:

* **dhcpd_service_enable**, *enable dhcpd after deploy*
* **dhcpd_service_restart**, *restart dhcpd after deploy*
* **dhcpd_apparmor_enable**, *enable apparmor after deploy*
* **dhcpd_apparmor_restart**, *restart apparmor after deploy*

Difference between global and subnet interface options
-----------------------------------------------------------
Global dhcpd_interfaces option makes listen on defined interfaces all subnets.
Interface per subnet definition allows listen as much subnets as you want.
Global dhcpd_interfaces option does not work on systemd distros (ArchLinux,
CentOS 7, Fedora), listen by default on interface with declared subnet. You
can rewrite systemd service, but is dirty. Instead this, describe interfaces in
configuration. Is modern and properly.

Global hosts
---------------
The role have global hosts and subnets definition:

```
dhcpd_global_hosts
dhcpd_global_subnets
```

And can be activated via `dhcpd_use_global: true`.

It is meant to define in `group_vars`, for example it can be management vlan,
or employees with laptops (traveling work) - new day in another office.

Extra
--------
* IP/MAC addresses, before deploy will be checked by Jinja with python-netaddr
lib;
* Base of subnet now defined as prefix, netmask of subnet will be defined via
Jinja filter;
* After successfully generation of dhcpd.conf it will be validated by dhcpd
itself - i.e. you can't broke your working configuration.

Role Variables
-----------------

The variables that can be passed to this role and a brief description about
them are as follows. These are all based on the configuration variables of the
DHCP server configuration.

```yaml
dhcpd_configure_apparmor: true|false
dhcpd_use_ansible_managed: true|false
dhcpd_add_deployed_timestamp: true|false
dhcpd_apparmor_enable: true|false
dhcpd_apparmor_restart: true|false
dhcpd_ddns_manage: true|false
dhcpd_install_package: true|false
dhcpd_service_enable: true|false
dhcpd_service_restart: true|false

# Basic configuration information
dhcpd_use_ansible_managed: 'true'
dhcpd_add_deployed_timestamp: 'true'
dhcpd_interfaces: eth0
dhcpd_common_domain: example.org
dhcpd_common_nameservers: ns1.example.org, ns2.example.org
dhcpd_common_default_lease_time: 600
dhcpd_common_max_lease_time: 7200
dhcpd_common_ddns_update_style: none
dhcpd_common_authoritative: true
dhcpd_common_log_facility: local7
dhcpd_common_options:
  - name: 'opt66'
    code: '66'
    type: 'string'
  - name: 'captive'
    code: '160'
    type: 'string'
dhcpd_common_parameters:
  - filename "pxelinux.0"
dhcpd_common_unknown_clients: allow|deny|ignore

# DDNS configuration
dhcpd_ddns_manage: true|false
dhcpd_ddns_client_updates: allow|ignore
dhcpd_ddns_updates: on|off
dhcpd_ddns_update_static_leases: on|off
dhcpd_ddns_update_style: standard|interim|none
dhcpd_ddns_keys:
  - the_key_name: the_key_value
dhcpd_ddns_zones:
  - name:example.org
    primary: 192.168.0.1
    key: a_key_name_from_dhcpd_ddns_keys_list

# Subnet configuration
dhcpd_subnets:
# Required variables example
- base: 192.168.1.0/24
# Full list of possibilities
- base: 192.168.10.0/24
  interface: vlan100
  range_start: 192.168.10.150
  range_end: 192.168.10.200
  routers: 192.168.10.1
  broadcast_address: 192.168.10.255
  domain_nameservers: 192.168.10.1, 192.168.10.2
  domain_name: example.org
  ntp_servers: pool.ntp.org
  default_lease_time: 3600
  max_lease_time: 7200
  pools:
  - range_start: 192.168.100.10
    range_end: 192.168.100.20
    rule: 'allow members of "foo"'
    parameters:
    - filename "pxelinux.0"
  - range_start: 192.168.110.10
    range_end: 192.168.110.20
    rule: 'deny members of "foo"'
  parameters:
  - filename "pxelinux.0"

# Fixed lease configuration
dhcpd_hosts:
- name: local-server
  mac_address: "00:11:22:33:44:55"
  fixed_address: 192.168.10.10
  default_lease_time: 43200
  max_lease_time: 86400
  parameters:
  - filename "pxelinux.0"

# Class configuration
dhcpd_classes:
- name: foo
  rule: 'match if substring (option vendor-class-identifier, 0, 4) = "SUNW"'
- name: 'CiscoSPA'
  rule: 'match if option vendor-class-identifier ~= "^(Cisco SPA)[0-9]+(G|)$"'
  options:
  - opt: 'opt66 "http://distrib.local/cisco.php?mac=$MAU&model=$PN"'
  - opt: 'time-offset 21600'

# Shared network configurations
dhcpd_shared_networks:
- name: shared-net
  interface: vlan100
  subnets:
  - base: 192.168.100.0/24
    routers: 192.168.10.1
    parameters:
    - filename "pxelinux.0"
    pools:
    - range_start: 192.168.100.10
      range_end: 192.168.100.20
      rule: 'allow members of "foo"'
      parameters:
      - filename "pxelinux.0"
    - range_start: 192.168.110.10
      range_end: 192.168.110.20
      rule: 'deny members of "foo"'

# Custom if else clause
dhcpd_ifelse:
- condition: 'exists user-class and option user-class = "iPXE"'
  value: 'filename "http://my.web.server/real_boot_script.php";'
  else:
  - value: 'filename "pxeboot.0";'
  - value: 'filename "pxeboot.1";'

```

Examples
--------------

1) Install DHCP server on interface eth0 with one simple subnet:

```yaml
- hosts: all
  roles:
  - role: dhcpd_server
    dhcpd_interfaces: eth0
    dhcpd_common_domain: example.org
    dhcpd_common_nameservers: ns1.example.org, ns2.example.org
    dhcpd_common_default_lease_time: 600
    dhcpd_common_max_lease_time: 7200
    dhcpd_common_ddns_update_style: none
    dhcpd_common_authoritative: true
    dhcpd_common_log_facility: local7
    dhcpd_subnets:
    - base: 192.168.10.0/24
      range_start: 192.168.10.150
      range_end: 192.168.10.200
      routers: 192.168.10.1
```

2) Install DHCP server with subnet per interface:

```yaml
- hosts: all
  roles:
  - role: dhcpd_server
    dhcpd_common_domain: example.org
    dhcpd_common_nameservers: ns1.example.org, ns2.example.org
    dhcpd_common_default_lease_time: 600
    dhcpd_common_max_lease_time: 7200
    dhcpd_common_ddns_update_style: none
    dhcpd_common_authoritative: true
    dhcpd_common_log_facility: local7
    dhcpd_subnets:
    - base: 192.168.10.0/24
      interface: vlan10
      range_start: 192.168.10.150
      range_end: 192.168.10.200
      routers: 192.168.10.1
    - base: 192.168.20.0/24
      interface: vlan20
      range_start: 192.168.20.150
      range_end: 192.168.20.200
      routers: 192.168.20.1
```

3) Install DHCP server with one subnet on interface vlan10 and with shared network on interface vlan20

```yaml
- hosts: all
  roles:
  - role: dhcpd_server
    dhcpd_common_default_lease_time: 600
    dhcpd_common_max_lease_time: 7200
    dhcpd_common_ddns_update_style: none
    dhcpd_common_authoritative: true
    dhcpd_common_log_facility: local7
    dhcpd_subnets:
    - base: 192.168.10.0/24
      interface: vlan10
      domain_nameserver: 192.168.10.1
      domain_name: example.local
      range_start: 192.168.10.150
      range_end: 192.168.10.200
      routers: 192.168.10.1
    dhcpd_shared_networks:
    - name: sharednet
      interface: vlan20
      subnets:
      - base: 10.7.0.0/24
        routers: 10.7.0.1
        domain_nameserver: 10.7.0.1
        domain_name: example.public0
        ntp_servers: 10.7.0.1
        pools:
        - range_start: 10.7.0.2
          range_end: 10.7.0.254
      - base: 10.8.0.0/24
        routers: 10.8.0.1
        domain_nameserver: 10.8.0.1
        domain_name: example.public1
        ntp_servers: 10.8.0.1
        pools:
        - range_start: 10.8.0.2
          range_end: 10.8.0.254
```
