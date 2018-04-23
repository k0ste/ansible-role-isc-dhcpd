# isc_dhcpd

This role installs and configures ISC dhcpd.

## Requirements for usage

* Ansible 2.4;
* [python-netaddr](//docs.ansible.com/ansible/playbooks_filters_ipaddr.html);

#### Ubuntu AppArmor

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

#### Package Installation

If you don't need install package, just deploy (save your time), disable package
installation by set variable `dhcpd_install_package: false`.

#### Handlers

You can control handlers behavior via variables:

* `dhcpd_service_enable` - enable dhcpd after deploy
* `dhcpd_service_restart` - restart dhcpd after deploy
* `dhcpd_apparmor_enable` - enable apparmor after deploy
* `dhcpd_apparmor_restart` - restart apparmor after deploy

#### Difference between global and subnet interface options

Global `dhcpd_interfaces` makes listen on defined interfaces all subnets.
Interface per subnet definition allows listen as much subnets as you want.
Global `dhcpd_interfaces` option does not work on systemd distros (ArchLinux,
CentOS 7, Fedora), listen by default on interface with declared subnet. You
can rewrite systemd service, but is dirty. Instead this, describe interfaces in
configuration. Is modern and properly.

#### Global hosts, subnets and classess

It is meant to define in `group_vars`, for example it can be management vlan,
or employees with laptops (traveling work) - new day in another office.

```
dhcpd_global_hosts
dhcpd_global_subnets
dhcpd_global_classes
```

And can be activated via `dhcpd_use_global: true`.
The syntax is match to non global definition.

#### Extra

* Deploy scenarios (ruled by `dhcpd_scenario` variable):
 - `validation` - perform hosts declarations validation (without deploy);
 - `deploy` - deploy without validation, this is default for speedup deploy;
 - `both` - deploy if validation pass;
* Base of subnet now defined as prefix, netmask of subnet will be defined via
Jinja filter;
* After successfully deploy of `dhcpd.conf` it will be validated by dhcpd
itself - i.e. you can't broke your working configuration.

#### Example configuration

```yaml
---
dhcpd_configure_apparmor: 'false'
dhcpd_use_ansible_managed: 'true'
dhcpd_add_deployed_timestamp: 'false'
dhcpd_apparmor_enable: 'false'
dhcpd_apparmor_restart: 'false'
dhcpd_ddns_manage: 'false'
dhcpd_install_package: 'false'
dhcpd_service_enable: 'true'
dhcpd_service_restart: 'true'
dhcpd_use_global: 'true'
dhcpd_scenario: 'deploy'

dhcpd_common_default_lease_time: '86400'
dhcpd_common_max_lease_time: '172800'
dhcpd_common_authoritative: 'true'
dhcpd_common_log_facility: 'local7'
# https://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml
dhcpd_common_options:
# TFTP Server Name (https://tools.ietf.org/html/rfc2132)
- name: 'opt66'
  code: '66'
  type: 'string'
# DHCP Captive-Portal (https://tools.ietf.org/html/rfc7710)
- name: 'captive'
  code: '160'
  type: 'string'
# PXE ConfigFile (https://tools.ietf.org/html/rfc5071)
- name: 'configfile'
  code: '209'
  type: 'text'
# PXE PathPrefix (https://tools.ietf.org/html/rfc5071)
- name: 'pathprefix'
  code: '210'
  type: 'text'
# PXE RebootTime (https://tools.ietf.org/html/rfc5071)
- name: 'reboottime'
  code: '211'
  type: 'unsigned integer 32'

dhcpd_global_classes:
# This saved in group_vars/all/dhcp_server.yaml for global access.
- name: 'Everywhere'
  rule: 'match hardware'
  subclass:
  - name: 'Notebook1'
    mac_address: 'd0:22:be:ce:7d:5c'
  - name: 'Notebook2'
    mac_address: 'd0:22:be:ac:e0:e1'

dhcpd_global_subnets:
- base: '198.18.1.0/26'
  interface: 'vlan666'
  routers: '198.18.1.1'
  domain_nameservers:
# One or many DNS nameservers.
  - '8.8.8.8'
  domain_name: 'example.com'
  ntp_servers:
# One or many NTP servers.
  - '0.ru.pool.ntp.org'
  - '1.ru.pool.ntp.org'
  pools:
  - range_start: '198.18.1.2'
    range_end: '198.18.1.62'
    options: # options only for this pool
    - key: 'captive'
      value: '"http://captive.portal/"'

dhcpd_classes:
# If client request math with this regex - answer with TFTP address and time
# offset.
- name: "CiscoSPA"
  rule: 'match if option vendor-class-identifier ~= "^(Cisco SPA)[0-9]+(G|)$"'
  options:
  - key: 'opt66'
    value: '"http://example.com/cisco.php?mac=$MAU&model=$PN"'
  - key: 'time-offset'
    value: '25200'

dhcpd_subnets:
# This subnet allow only mac addresses of subclasses of 'Everywhere' class.
- base: '198.19.1.0/26'
  interface: 'vlan100'
  routers: '198.19.1.1'
  interface_mtu: '9000'
  domain_nameservers:
  - '198.19.1.1'
  domain_name: 'example.com'
  ntp_servers:
  - '198.19.1.1'
  pools:
  - range_start: '198.19.1.2'
    range_end: '198.19.1.60'
    rule: 'allow members of "Everywhere"'
## Usual subnet. Allow only static clients (declared in dhcpd_hosts or
## dhcpd_global_hosts). If host matched with class - special options are offered.
## Resolve hostnames from DNS.
- base: '192.168.1.0/24'
  interface: 'vlan200'
  routers: '192.168.1.1'
  domain_nameservers:
  - '192.168.1.1'
  domain_name: 'example.com'
  ntp_servers:
  - '192.168.1.1'
  unknown_clients: 'deny'
  get_lease_hostnames: 'true'
  pools:
  - range_start: '192.168.1.1'
    range_end: '192.168.1.1'
    rule: 'allow members of "CiscoSPA"'

dhcpd_shared_networks:
## Two subnets in one VLAN.
- name: 'example'
  interface: 'vlan300'
  subnets:
  - base: '172.16.201.0/26'
    routers: '172.16.201.1'
    domain_nameservers:
    - '172.16.201.1'
    domain_name: 'example.com'
    ntp_servers:
    - '0.ru.pool.ntp.org'
    - '1.ru.pool.ntp.org'
  - base: '172.16.202.0/26'
    routers: '172.16.202.1'
    domain_nameservers:
    - '172.16.202.1'
    domain_name: 'example.com'
    ntp_servers:
    - '0.ru.pool.ntp.org'
    - '1.ru.pool.ntp.org'

dhcpd_groups:
# Group of hosts
- name: 'foo'
# This option allow to generate host-name option from host name definition, when
# host_name actually not defined.
  host_name_from_name: 'true'
  filename: '/arch/boot/syslinux/lpxelinux.0'
# The next_server statement specifies the IP address of the TFTP server from
# which a client can download the boot loader file.
  next_server: '10.10.10.10'
  options: # dhcp options only for this group
  - key: 'dhcp-parameter-request-list'
    value: 'concat(option dhcp-parameter-request-list,d1,d2,d3)'
    equal_sign: 'true' # add '=' char between key and value, false by default
  - key: 'configfile'
    value: '"boot/syslinux/archiso.cfg"'
    equal_sign: 'true'
  - key: 'pathprefix'
    value: '"/arch/"'
    equal_sign: 'true'
  - key: 'reboottime'
    value: '10'
    equal_sign: 'true'
  hosts:
  - name: 'Bob'
    mac_address: '00:11:22:33:44:55'
  - name: 'Alice'
    mac_address: '00:11:22:33:44:66'

dhcpd_hosts:
# Static hosts.
- name: 'host1'
  mac_address: 'dc:9f:db:1a:4d:c7'
  fixed_address: '192.168.1.2'
- name: 'host2'
  mac_address: 'dc:9f:db:1a:1a:05'
  fixed_address: '192.168.1.3'
# Static host with hostname and domain name.
- name: 'host3'
  mac_address: 'dc:9f:db:1a:1b:1c'
  fixed_address: '192.168.1.4'
  host_name: 'testhost'
  domain_name: 'testdomain.com'
# Static VoIP, for class matching.
- name: 'voip1'
  mac_address: 'dc:9f:db:1a:4d:c8'
  fixed_address: '192.168.1.4'
- name: 'voip2'
  mac_address: 'dc:9f:db:1a:1a:06'
  fixed_address: '192.168.1.5'
```
