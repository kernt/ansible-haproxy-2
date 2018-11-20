# Ansible HAProxy Role

Installs HAProxy on RedHat/CentOS GNU/Linux servers with high availability support.

**Note**: This role supports HAProxy version `1.5`. Future versions may require some rework.

## Requirements

None.

## Dependencies

None.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

### HAProxy configuration

- **`haproxy_processes`**: Number of HAProxy processes (default: `1`).

- **`haproxy_socket`**: The socket through which HAProxy can communicate for admin purposes or statistics (default: `/var/lib/haproxy/stats`). It will be suffixed with a dash and HAProxy process ordinal number starting from `1`.

- **`haproxy_chroot`**: The jail directory where chroot() will be performed before dropping privileges (default: `/var/lib/haproxy`).

- Default user and group under which HAProxy should run:

```
    haproxy_user: haproxy
    haproxy_group: haproxy
```

- **`haproxy_enable_stats`**: HAProxy stats URL is disabled by default. To enable stats, set `haproxy_enable_stats` to `true` (default: `false`).

- **`haproxy_stats_username / haproxy_stats_password`**: Stats access would be protected by login/pass. Set the role parameters to enforce authentication (default: empty).

- Default HAProxy frontend configuration directives:

```
  haproxy_frontend_mode: 'http'
```

- Default HAProxy backend configuration directives:

```
  haproxy_backend_mode: 'http'
  haproxy_backend_balance_method: 'roundrobin'
  haproxy_backend_httpchk: 'HEAD / HTTP/1.1\r\nHost:\{\{ ansible_fqdn \}\}'
```

- A list of backend servers (name and address) to which HAProxy will distribute requests (default: `[]`). Example:

```
  haproxy_backend_servers:
    - name: srv-1
      address: 192.168.0.1:80
    - name: srv-2
      address: 192.168.0.2:80
```

- A list of extra global variables to add to the global configuration section inside `haproxy.cfg` (default: `[]`). Example:

```
  haproxy_extra_global_vars:
    - 'ssl-default-bind-ciphers ABCD+KLMJ:...'
    - 'ssl-default-bind-options no-sslv3'
```

### Keepalived configuration

- **`haproxy_notification_email`**: E-mail address to send notifications to (default: `root`).

- **`haproxy_notification_email_from`**: Sender e-mail address (default: `haproxy@fqdn`).

- **`haproxy_enable_ha`**: Configure HA for HAProxy instances using Keepalived (default: `false`).

- **`haproxy_keepalived_vrrps`**: List of VRRPs for Keepalived (example below):

```
haproxy_keepalived_vrrps:
- name: "VI_1"			# A unique, by NIC, virtual instance name to configure by Keepalived
  interface: "eth0"		# Interface to bind to
  state: "MASTER"		# The virtual instance type. Could be "MASTER|BACKUP|FAULT"
  router_id: 51			# Router ID for VRRP unique within interface context
  priority: 200			# The instance priority 1-255. For backups this must be lower than master's
  vips:	    			# List of VIPs and their netmasks and options
    - address: 192.168.0.1
      mask: 32
    - address: 192.168.0.2
      mask: 32
```

## Usage

```yaml

    - hosts: balancer
      become: yes
      roles:
        - role: ansible-haproxy
```

You may manually need to allow setgid/setuid rights for keepalived due to lacking SELinux definitions for it:
```
module my-keepalived 1.0;

require {
        type keepalived_t;
        class capability { setgid setuid };
}

#============= keepalived_t ==============

allow keepalived_t self:capability setgid;
allow keepalived_t self:capability setuid;
```
## License

MIT

## Author Information

This role was created by [Ahmed
Bessifi](https://www.linkedin.com/in/abessifi), a DevOps enthusiast
and further developed by Tom Söderlund.
