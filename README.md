# Fail2ban

[![Build Status](https://drone.osshelp.ru/api/badges/ansible/fail2ban/status.svg)](https://drone.osshelp.ru/ansible/fail2ban)

Role which installs and configures Fail2ban.

## Role usage

The role should be used after other roles installing software which needs protection.

## Deploy example (do not copy blindly!)

```yaml
roles:
    - role: fail2ban
      fail2ban_ignores_ips: ['10.0.0.0/8']
      fail2ban_enable_ignorecommand: true
      fail2ban_custom_ipset_lists: [oss-v4-main, oss-v6-main]
      fail2ban_recidive_ignore_jails: [some-jail, another-jail]
      fail2ban_filters: [
        { name: nginx-req-limits,
          failregex: [ '^\s*\[error\] \d+#\d+: \*\d+ limiting requests, excess: [\d\.]+ by zone "[^"]+", client: <HOST>' ],
          ignoreregex: []
        },
        { name: nginx-con-limits,
          failregex: [ '^\s*\[error\] \d+#\d+: \*\d+ limiting connections by zone "[^"]+", client: <HOST>' ],
          ignoreregex: []
        },
        { name: multiple-regexps-example,
          failregex: [ 'some-fail-regexp1', 'some-fail-regexp2' ],
          ignoreregex: [ 'some-ignore-regexp1', 'some-ignore-regexp2' ]
        }
      ]
      fail2ban_sshd:
        maxretry: 5
        bantime: 3600
        findtime: 600
      fail2ban_services: [
        { name: nginx-req-limits,
          filter: nginx-req-limits,
          port: 'http,https',
          logpath: '/var/log/nginx/*error.log',
          bantime: 600,
          findtime: 300,
          maxretry: 5
        },
        { name: nginx-con-limits,
          filter: nginx-con-limits,
          port: 'http,https',
          logpath: '/var/log/nginx/*error.log',
          bantime: 600,
          findtime: 300,
          maxretry: 5
        }
      ]
      fail2ban_containers: [
        { name: web-prod,
          logpath: /tmp/web-prod.log
        },
        { name: backend-prod,
          logpath: /tmp/backend-prod.log,
          bantime: 9000
        }
      ]
```

## About available parameters

### Main params

| Param | Default | Description |
| -------- | -------- | -------- |
| `fail2ban_setup` | `full` | See [OSSHelp KB article](https://oss.help/kb4895) |
| `fail2ban_defaults` | see defaults/main.yml | controls default bantime, findtime and maxretry params |
| `fail2ban_installation_type` | `repository` | Change this to `osshelp` if you need to install newer version of fail2ban on xenial from our [testing repo](https://oss.help/kb697). Versions older than [0.10.0](https://github.com/fail2ban/fail2ban/releases/tag/0.10.0) do not support IPv6. |
| `fail2ban_ignores_ips` | - | controls list of IP's to ignore (see ignoreip fail2ban param) |
| `fail2ban_alerts` | see defaults/main.yml | control on alert-sending, just in case if we'll need it anywhere |
| `fail2ban_filters` | - | describes the filters to create, more details below |
| `fail2ban_services` | - | describes the custom jails to create, more details below |
| `fail2ban_containers` | - | params for sshd-jails for LXC containers, more details below |
| `fail2ban_blocktype` | `DROP` | we want to drop packets instead of rejecting them. Do not change this without a reason |
| `fail2ban_role_test_mode` | - | could be used for test purposes only, do not use it in production playbooks |
| `fail2ban_enable_ignorecommand` | `false` | enables ignorecommand options in jail.local and checking script generation |
| `fail2ban_default_ipset_lists` | `[oss-v4, oss-v6]` | describes the default ipset lists for checking script |
| `fail2ban_custom_ipset_lists` | `[]` | describes the custom ipset lists for checking script |
| `fail2ban_dummy_logs` | `false` | Whether to create dummy logs to avoid service failing to start due to absence of any jail logs. |
| `fail2ban_dummy_log_path` | `/var/log/fail2ban-dummy.log` | Path to dummy log (will be automatically created). |
| `fail2ban_recidive_ignore_jails` | `[]` | List of jails that need to be ignored by recidive jail. |

### Jail generation

#### SSH jails

Our common sshd jail is generated automatically if openssh-server package was found during role setup. By default it will look like this:

```plaintext
[sshd]
enabled    = true
ports      = 22
maxretry   = 3
bantime    = 7200
findtime   = 1800
```

If needed, default params for this jail could be overwritten from playbook with **fail2ban_sshd** variable, for example:

#### SSH jails for LXC containers

```yaml
      fail2ban_sshd:
        maxretry: 5
        bantime: 3600
        findtime: 600
```

Jails for LXC containers are being created only if **fail2ban_containers** list is not empty. You can control any param of this jail from playbook, for example:

```yaml
      fail2ban_containers: [
        { name: web-prod,
          logpath: /mnt/data/containers/web-prod/var/log/auth.log.log
        },
        { name: backend-prod,
          logpath: /mnt/data/containers/backend-prod/var/log/auth.log,
          bantime: 9000,
          findtime: 600
        }
      ]
```

Make sure to set "logpath" variable, role can't guess the correct path by itself!

#### Custom jails

Could be created with **services** list, usage example:

```yaml
      fail2ban_services: [
        { name: nginx-req-limits,
          filter: nginx-req-limits,
          port: 'http,https',
          logpath: '/var/log/nginx/*error.log',
          bantime: 600,
          findtime: 300,
          maxretry: 5
        },
        { name: nginx-con-limits,
          filter: nginx-con-limits,
          port: 'http,https',
          logpath: '/var/log/nginx/*error.log',
          bantime: 600,
          findtime: 300,
          maxretry: 5
        }
      ]
```

You can describe any key-value pairs here. **name** is mandatory for jail creation.

### Custom filters creation

Can be done with **filters** variable. For example:

```yaml
      fail2ban_filters: [
        { name: nginx-req-limits,
          failregex: [ '^\s*\[error\] \d+#\d+: \*\d+ limiting requests, excess: [\d\.]+ by zone "[^"]+", client: <HOST>' ],
          ignoreregex: []
        },
        { name: nginx-con-limits,
          failregex: [ '^\s*\[error\] \d+#\d+: \*\d+ limiting connections by zone "[^"]+", client: <HOST>' ],
          ignoreregex: []
        },
        { name: multiple-regexps-example,
          failregex: [ 'some-fail-regexp1', 'some-fail-regexp2' ],
          ignoreregex: [ 'some-ignore-regexp1', 'some-ignore-regexp2' ]
        }
      ]
```

You can describe any key-value pairs here. **name** is mandatory for filter creation.

### Checking ip in ipset lists

Fail2ban can use external command to dynamically check if IP should be ingored. Option fail2ban_enable_ignorecommand enables it. External script will check the IP’s presence in every ipset list from fail2ban_default_ipset_lists and fail2ban_custom_ipset_lists. If IP will be found in any list given - fail2ban will ignore it, if not - IP will be placed in appropriate jail.

## Useful links

- [Official wiki](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [Documentation on ubuntu.ru](https://help.ubuntu.ru/wiki/fail2ban)
- [Our article about manual Fail2ban setup](https://oss.help/kb503)
- [Our article about configuration for LXC containers](https://oss.help/kb245)

## TODO

- update readme
- remove syslog user creation (leave in test mode only maybe)
- move names and paths from check_ip_script template to vars
