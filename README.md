# network_interface


_WARNING: This role can be dangerous to use. If you lose network connectivity
to your target host by incorrectly configuring your networking, you may be
unable to recover without physical access to the machine._

This roles enables users to configure various network components on target
machines. The role can be used to configure:

- Ethernet interfaces
- Bridge interfaces
- Bonded interfaces
- VLAN tagged interfaces
- Network routes

## Requirements


This role requires Ansible 1.4 or higher, and platform requirements are listed
in the metadata file.

## Role Variables


The variables that can be passed to this role and a brief description about
them are as follows:

| Variable | Required | Default | Comments |
|-------------------------------------|----------|-----------|---------|
| `network_pkgs` | No | `[]` | Typically needed packages like selinux, bridge-utils, ifenslave and iproute |
| `network_ether_interfaces` | No | `[]` | The list of ethernet interfaces to be added to the system. |
| `network_bridge_interfaces` | No | `[]` | The list of bridge interfaces to be added to the system. |
| `network_bond_interfaces` | No | `[]` | The list of bonded interfaces to be added to the system. |
| `network_vlan_interfaces` | No | `[]` | The list of vlan interfaces to be added to the system. |
| `network_check_packages` | No | `true` | Install packages listed in network_pkgs. |
| `network_allow_service_restart` | No | `true` | Whether interfaces/networking should get reconfigured and restarted. |
| `network_modprobe_persist` | No | `true` | Persisting module loading. |
| `network_configured_interfaces_only` | No | `false` | Removes interfaces not configured over this role entirely when enabled. |
| `network_interface_file_prefix` | No | `ifcfg-` | The prefix for interface configuration files. |
| `network_interface_file_postfix` | No | `` | The postfix for interface configuration files. |

Note: The values for the list are listed in the examples below.

## Examples

Debian (not RedHat) network configurations can optionally use CIDR notation for IPv4 addresses instead of specifying the address and subnet mask separately. It is required to use CIDR notation for IPv6 addresses on Debian.

IPv4 example with CIDR notation:
```
      cidr: 192.168.10.18/24
      # OPTIONAL: specify a gateway for that network, or auto for network+1
      gateway: auto
```
IPv4 example with classic IPv4:
```
      address: 192.168.10.18
      netmask: 255.255.255.0
      network: 192.168.10.0
      broadcast: 192.168.10.255
      gateway: 192.168.10.1
```
If you want to use a different MAC Address for your Interface, you can simply add it.
```
      hwaddress: aa:bb:cc:dd:ee:ff
```
On some rare occasion it might be good to set whatever option you like. Therefore it
is possible to use
```
      options:
          - "up /execute/my/command"
          - "down /execute/my/other/command"
```
and the IPv6 version
```
      ipv6_options:
          - "up /execute/my/command"
          - "down /execute/my/other/command"
```

1) Configure eth1 and eth2 on a host with a static IP and a dhcp IP. Also
define static routes and a gateway.
```

- hosts: myhost
  roles:
    - role: network
      network_ether_interfaces:
       - device: eth1
         bootproto: static
         cidr: 192.168.10.18/24
         gateway: auto
         route:
          - network: 192.168.200.0
            netmask: 255.255.255.0
            gateway: 192.168.10.1
          - network: 192.168.100.0
            netmask: 255.255.255.0
            gateway: 192.168.10.1
       - device: eth2
         bootproto: dhcp
```
Note: it is not required to add routes, default route will be added automatically.

2) Configure a bridge interface with multiple NICs added to the bridge.
```

- hosts: myhost
  roles:
    - role: network
      network_bridge_interfaces:
       -  device: br1
          type: bridge
          cidr: 192.168.10.10/24
          bridge_ports: [eth1, eth2]

          # Optional values
          bridge_ageing: 300
          bridge_bridgeprio: 32768
          bridge_fd: 15
          bridge_gcint: 4
          bridge_hello: 2
          bridge_maxage: 20
          bridge_maxwait: 0
          bridge_pathcost: "eth1 100"
          bridge_portprio: "eth1 128"
          bridge_stp: "on"
          bridge_waitport: "5 eth1 eth2"
```

Note: Routes can also be added for this interface in the same way routes are
added for ethernet interfaces.

3) Configure a bond interface with an "active-backup" slave configuration.
```

- hosts: myhost
  roles:
    - role: network
      network_bond_interfaces:
        - device: bond0
          address: 192.168.10.128
          netmask: 255.255.255.0
          bond_mode: active-backup
          bond_slaves: [eth1, eth2]

          # Optional values
          bond_miimon: 100
          bond_lacp_rate: slow
          bond_xmit_hash_policy: layer3+4
```

4) Configure a bonded interface with "802.3ad" as the bonding mode and IP
address obtained via DHCP.
```

- hosts: myhost
  roles:
    - role: network
      network_bond_interfaces:
        - device: bond0
          bootproto: dhcp
          bond_mode: 802.3ad
          bond_miimon: 100
          bond_slaves: [eth1, eth2]
          bond_ad_select: 2
```

5) Configure a VLAN interface with the vlan tag 2 for an ethernet interface
```

- hosts: myhost
  roles:
    - role: network
      network_ether_interfaces:
       - device: eth1
         bootproto: static
         cidr: 192.168.10.18/24
         gateway: auto
      network_vlan_interfaces:
       - device: eth1.2
         bootproto: static
         cidr: 192.168.20.18/24
```

6) All the above examples show how to configure a single host, The below
example shows how to define your network configurations for all your machines.

Assume your host inventory is as follows:

### /etc/ansible/hosts
```
[dc1]
host1
host2
```
Describe your network configuration for each host in host vars:

### host_vars/host1
```
    network_ether_interfaces:
           - device: eth1
             bootproto: static
             address: 192.168.10.18
             netmask: 255.255.255.0
             gateway: 192.168.10.1
             route:
              - network: 192.168.200.0
                netmask: 255.255.255.0
                gateway: 192.168.10.1
    network_bond_interfaces:
            - device: bond0
              bootproto: dhcp
              bond_mode: 802.3ad
              bond_miimon: 100
              bond_slaves: [eth2, eth3]
```
### host_vars/host2
```
network_ether_interfaces:
       - device: eth0
         bootproto: static
         address: 192.168.10.18
         netmask: 255.255.255.0
         gateway: 192.168.10.1
```

7) If resolvconf package should be used, it is possible to add some DNS configurations
```
dns-nameserver: [ "8.8.8.8", "8.8.4.4" ]
dns-search: "search.mydomain.tdl"
dns-domain: "mydomain.tdl"
```

8) You can add IPv6 static IP configuration on Ethernet, Bond or Bridge interfaces
```
ipv6_address: "aaaa:bbbb:cccc:dddd:dead:beef::1/64"
ipv6_gateway: "aaaa:bbbb:cccc:dddd::1"
```

Create a playbook which applies this role to all hosts as shown below, and run
the playbook. All the servers should have their network interfaces configured
and routed updated.

```

- hosts: all
  roles:
    - role: network
```

9) This role can also optionally add network interfaces to firewalld zones. The
core firewalld module (http://docs.ansible.com/ansible/latest/firewalld_module.html)
can perform the same function, so if you make use of both modules then your
playbooks may not be idempotent.  Consider this case, where only the firewalld
module is used:

  * network_interface role runs; with no `firewalld_zone` host var set then any
    ZONE line will be removed from ifcfg-*
  * `firewalld` module runs; adds a `ZONE` line to ifcfg-*
  * On the next playbook run, the network_interface role runs and removes the
    ZONE line again, and so the cycle repeats.

In order for this role to manage firewalld zones, the system must be running a
RHEL based distribution, and using NetworkManager to manage the network
interfaces.  If those criteria are met, the following example shows how to add
the eth0 interface to the public firewalld zone:
```
       - device: eth0
         bootproto: static
         address: 192.168.10.18
         netmask: 255.255.255.0
         gateway: 192.168.10.1
         firewalld_zone: public
```
Note: Ansible needs network connectivity throughout the playbook process, you
may need to have a control interface that you do *not* modify using this
method while changing IP Addresses so that Ansible has a stable connection
to configure the target systems. All network changes are done within a single
generated script and network connectivity is only lost for few seconds.


## Dependencies

`python-netaddr`

## License


BSD

## Author Information

This project was originally created by [Benno Joy](https://github.com/bennojoy/network_interface).

Debian upgrades by:

* Martin Verges (croit, GmbH)
* Eric Anderson (Avi Networks, Inc.)

RedHat upgrades by:

* Eric Anderson (Avi Networks, Inc.)
* Luke Short (Red Hat, Inc.)
* Wei Tie, (Cisco Systems, Inc.)

The full list of contributors can be found [here](https://github.com/MartinVerges/ansible.network_interface/graphs/contributors).
