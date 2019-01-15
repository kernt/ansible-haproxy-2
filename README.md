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

- **`haproxy_firewalld_name`**: Name of automatically generated service for firewalld (default: `haproxy`)

- **`haproxy_firewalld_file`**: Name of automatically generated temporary XML file to create HAProxy service for firewalld (default: `{{ haproxy_firewalld_name }}.xml`)

- **`haproxy_sysctl_file`**: Name of automatically generated sysctl file for HAProxy (default: `/etc/sysctl.d/z0-haproxy.conf`)

- **`haproxy_logrotate_file`**: Name of automatically generated logrotate file for HAProxy (default: `/etc/logrotate.d/haproxy`)

- **`haproxy_rsyslog_file`**: Name of automatically generated rsyslog file for HAProxy (default: `/etc/rsyslog.d/haproxy.conf`)

- **`haproxy_extra_global_vars`**: A list of extra global variables to add to the global configuration section inside `haproxy.cfg` (default: `[]`).

- **`haproxy_bind_options`**: Argument to ssl-default-bind-options in the global configuration section (default: `no-sslv3 no-tls-tickets`)

- **`haproxy_bind_ciphers`**: Argument to ssl-default-bind-ciphers in the global configuration section (default: `ECDH+AESGCM:ECDH+CHACHA20:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS`)

- **`haproxy_server_options`**: Argument to ssl-default-server-options in the global configuration section (default: `no-sslv3`)

- **`haproxy_server_ciphers`**: Argument to ssl-default-server-ciphers in the global configuration section (default: `ECDH+AESGCM:ECDH+CHACHA20:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS`)

- **`haproxy_processes`**: Number of HAProxy processes (default: `1`).

- **`haproxy_socket`**: The socket through which HAProxy can communicate for admin purposes or statistics (default: `/var/lib/haproxy/stats`). It will be suffixed with a dash and HAProxy process ordinal number starting from `1`.

- **`haproxy_chroot`**: The jail directory where chroot() will be performed before dropping privileges (default: `/var/lib/haproxy`).

- Default user and group under which HAProxy should run:

```
    haproxy_user: haproxy
    haproxy_group: haproxy
```

- **`haproxy_enable_stats`**: HAProxy stats URL is disabled by default. To enable stats, set `haproxy_enable_stats` to `true` (default: `false`).

- **`haproxy_stats_port`**: HAProxy stats port to listen to (default: `1936`).

- **`haproxy_stats_username / haproxy_stats_password`**: Stats access would be protected by login/pass. Set the role parameters to enforce authentication (default: empty).

- **`haproxy_frontend_mode`**: Default HAProxy frontend mode (default: `http`)

- **`haproxy_backend_mode`**: Default HAProxy backend mode (default: `http`)

- **`haproxy_backend_balance_method`**: Default backend balance method (default: `roundrobin`)

- **`haproxy_frontends`**: A list of frontends which HAProxy will bind for servicing clients (default: `[]`). Example:

```
  haproxy_frontends:
  - name: fe_http
    binds:
    - "192.168.0.1:80"
    options:
    - httplog
    - forwardfor
    - http-ignore-probes
    default_backend: be_http

  - name: fe_https
    mode: tcp
    binds:
    - "192.168.0.1:443"
    options:
    - tcplog
    default_backend: be_https
```

- **`haproxy_backends`**: A list of backends to which HAProxy will distribute requests (default: `[]`). Example:
```
  haproxy_backends:
  - name: be_http
    stick: "on src table be_https"
    options:
    - httpchk
    settings:
    - http-check disable-on-404
    - http-request set-header X-Forwarded-Port %[dst_port]
    - http-request add-header X-Forwarded-Proto http
    servers:
    - name: srv01_http
      address: "192.168.1.1:80"
      options: check
    - name: srv02_http
      address: "192.168.1.2:80"
      options: check

  - name: be_https
    mode: tcp
    stick_table: "type ip size 200k expire 30m"
    stick: on src
    options:
    - tcplog
    - ssl-hello-chk
    settings:
    - http-request add-header X-Forwarded-Proto https
    servers:
    - name: srv01_https
      address: "192.168.1.1:443"
      options: check verify none
    - name: srv02_https
      address: "192.168.1.2:443"
      options: check verify none
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
```

## Usage

```yaml

- hosts: balancer
  serial: "50%"
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

```bash
> checkmodule -M -m -o my-keepalived.mod my-keepalived.te
> semodule_package -o my-keepalived.pp -m my-keepalived.mod
> sudo semodule -i my-keepalived.pp
```

## License

MIT

## Author Information

This role was created by [Ahmed
Bessifi](https://www.linkedin.com/in/abessifi), a DevOps enthusiast
and further developed by Tom Söderlund.
