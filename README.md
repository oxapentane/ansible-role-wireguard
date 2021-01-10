Wireguard ansible role
======================

Oversimplistic (at this point) role to set up barebones (see above) wireguard network among inventory machines. Part of the point was me learning ansible, so if you google you'll definetly will find better solutions. This role main purpose is to create a VPN where nodes that behind NATs can talk to each other via the nodes with public IP. You can find an example below, with explanation how the templating hack works to achieve that.

HOWTO
-----
Set appropriate variables in your `inventory.yaml` (or hostvars, whereever you keep your host variables), add `wireguard` role, hit big red button, and you should be good to go.

### Supported systems

At the moment the setup tasks are only available for CentOS and Fedora.

### Example Config

`inventory.yaml`
```yaml
wg_lab:
  vars:
    wg_iface: "wg-lab"
  hosts:
    vps.examle.com:
      wg_address: "10.99.0.1/24"
      wg_allowed_IPs: "10.99.0.1/24"
      wg_endpoint: "vps.example.com"
      wg_preup:
        - sysctl net.ipv4.ip_forward=1
        - "firewall-cmd --add-port={{ wg_listen_port }}/udp"
      wg_postup:
        - iptables -A FORWARD -i %i -j ACCEPT
        - iptables -A FORWARD -o %i -j ACCEPT
      wg_predown:
        - iptables -D FORWARD -i %i -j ACCEPT
        - iptables -D FORWARD -o %i -j ACCEPT
      wg_persistent_keepalive: "25"
    client1:
      wg_address: "10.99.0.2/32"
      wg_allowed_IPs: "10.99.0.2/32"
      wg_persistent_keepalive: "25"
    localhost:
      ansible_connection: local
      wg_address: "10.99.0.10/32"
      wg_allowed_IPs: "10.99.0.10/32"
      wg_persistent_keepalive: "25"
```

`playbook.yaml`
```yaml
---
- hosts: wg_lab
  roles:
    - wireguard
```

This is an example setup with one public peer (vps) and two peers behind NAT. On the vps preup commands ensure that ip forwaring is enabled, and that the wireguard port is open on public interface. Postup commands enable the routing between peers on the interface.

Since wireguard automatically adds routes to all the peers, this hack is present in the config file template:

```jinja2
{% for peer in ansible_play_hosts %}
{% if peer != inventory_hostname and (wg_endpoint is defined or hostvars[peer].wg_endpoint is defined) %}

[Peer]
# all the peer config magic

{% endif %}
{% endfor %}
```

So, the for statement loops over all the hosts in the play, and if selects hosts that are:
- not current host (duh!)
- if current host has an endpoint defined (aka this host has a public IP) we list all the other peers (so this host acts like a vpn "server")
- if current host has no endpoint ("client"), only list peers with endpoint defined (because other peers will be unreachable)


Variables
---------

for default values see `defaults/main.yaml`

host variables:

variable                             | description
:------------------------------------|:----
`wg_conf_dir`                        | location of configuration directory
`wg_iface`                           | interface name
`wg_conf_dir_mode`                   | mode of the config directory
`wg_conf_mode`                       | mode of the config file

Vars that directly translate to the wireguard configuration file
- `wg_address`
- `wg_listen_port`
- `wg_DNS`
- `wg_allowed_IPs`
- `wg_endpoint`
- `wg_persistent_keepalive`
- `wg_preup`
- `wg_postup`
- `wg_predown`
- `wg_postdown`

internal variables:
- `wg_reg*`: results of `register:`
- `wg_public_key`, `wg_private_key`

TODO
----
- [ ] add support for IPv6 in the template (ideally both dual stack (4/6) and IPv6 only)
- [ ] add support for missing wireguard config entries
- [ ] add MacOS support
- [ ] add a way to run only some tasks in the role (if you need to just reconfigure, you don't need to go through all the check/install/setup parts)
