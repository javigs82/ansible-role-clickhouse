# Ansible Clickhouse

This role is in charge of configuring & installing a clickhouse cluster with N shard and M replicas in `centos7`.

**Clickhouse-cluster** is built on top of hostname, so ensure hostname is properly set for
being used in [defaults](./defaults/main.yml) 

## Requirements

Please install following package to use `molecule` as TDD tool

```sh

python3 -m pip install --user "molecule[vagrant,lint]"
pip3 install testinfra


```

where `vagrant` is the driver and `testinfra` is the testing provider.

This solution is based on the ability of resolve hostname in a private dns server.
So, assuming vagrant does not provide any dns solution, the following software is installed in prepare.yml,
providing vagrant infrastructure with internal dns resolver

```yml

---
- name: Prepare
  hosts: all
  tasks:
    - name: Install epel-release
      yum:
        name: epel-release
        state: present
  
    - name: Install nss-mdns
      yum:
        name: nss-mdns
        state: present

    - name: Stop service cron on debian, if running
      systemd:
        name: avahi-daemon
        state: started

```

with `nss-mdns` and `avahi`, vagrant is able to resolve dns just like **<hostname>.local**

**Notice** that DNS resolution supposes to be more sophisticated in a real world use case.

## Design

 - Download: From yandex rpm repository. Downgrade is allowed by `clickhouse_allow_downgrade` property.
 - Config: Ensure `clickhouse` group & user. Ensure paths and config files
 - Install: Download and install by yum
 - Users: Dynamic list to manage users. Password management is not implemented
 - RBAC: TO BE IMPLEMENTED

### Cluster configuration

In order to "mount" the cluster, `hostname` takes an important consideration.

Cluster is defined in a yaml structure like:

> shardN.replicaN.local

where **shardN.replicaN** is the hostname and **local** is the domain to be resolved. Notice that this role implements vagrant-molecule and use special configuration to use service discovery. [Click here for more info](./molecule/default/prepare.yml)

## Role Variables

Check variables in [defaults](./defaults/main.yml)

### Version

Use these variable to set up display_name, version and allow downgrade

```yml

clickhouse_version: "20.9.5.5"
clickhouse_allow_downgrade: false
clickhouse_display_name: "mycluster"

```

### Yum Support

Use these variables to set up yum repository

```yml

clickhouse_supported: yes
clickhouse_repo: "https://repo.clickhouse.tech/rpm/stable/x86_64/"
clickhouse_repo_key: https://repo.clickhouse.tech//CLICKHOUSE-KEY.GPG
clickhouse_package:
  - "clickhouse-client-{{ clickhouse_version }}"
  - "clickhouse-common-static-{{ clickhouse_version }}"
  - "clickhouse-server-{{ clickhouse_version }}"

```

### Clickhouse Configuration

With those variables, main configuration is set up.

```yml

clickhouse_config:
  max_connections: 2048
  keep_alive_timeout: 3
  max_concurrent_queries: 100
  uncompressed_cache_size: 8589934592
  mark_cache_size: 5368709120
  builtin_dictionaries_reload_interval: 3600
  max_session_timeout: 3600
  default_session_timeout: 60
  mlock_status: false

```

### Cluster Configuration

Cluster configuration relies on hostname and with that, 
[macros.xml](./templates/macros.xml.j2) is dynamically built

```yml

clickhouse_cluster:
  shard01:
    - { host: "ch-shard01-replica01.local", port: "{{ clickhouse_tcp_port }}" }
    - { host: "ch-shard01-replica02.local", port: "{{ clickhouse_tcp_port }}" }
  shard02:
    - { host: "ch-shard02-replica01.local", port: "{{ clickhouse_tcp_port }}" }
    - { host: "ch-shard02-replica02.local", port: "{{ clickhouse_tcp_port }}" }

```

### Networking

These variable are use to configure networking

```yml

clickhouse_http_port: 8123
clickhouse_https_port: 8443
clickhouse_tcp_port: 9000
clickhouse_tcp_secure_port: 9440
clickhouse_interserver_http: 9009

clickhouse_listen_host_default:
  - "::1"
  - "0.0.0.0"
clickhouse_listen_host_custom: []
clickhouse_listen_host: "{{ clickhouse_listen_host_default + clickhouse_listen_host_custom }}"

```

**Notice** that `clickhouse_listen_host` must allow ch members listening

### Users

Use these vars to customize users. In order to remove a user use clickhouse config attributes:
https://clickhouse.tech/docs/en/operations/configuration-files/

```yml

clickhouse_user_list:
  - { user_name: "reader",
      profile: "readonly",
      password: "r3ad3R",
      networks: ["::/0"],
      quota: "default",
      attribute: "replace" }
  - { user_name: "jaimito",
      profile: "default",
      password: "J4imit0",
      networks: ["::/0"],
      quota: "default",
      attribute: "replace"  }
  - { user_name: "default",
      profile: "default",
      networks: ["::1", "127.0.0.1"],
      quota: "default",
      attribute: "replace" }
  - { user_name: "luis",
      profile: "default",
      password: "enr1Que",
      networks: ["::1", "127.0.0.1"],
      quota: "default",
      attribute: "remove" }

```

### Zookeeper

In order to use zookeeper to synchronize the cluster, set up zookeeper servers as following

```yml

clickhouse_zookeeper_nodes:
  - { host: "zookeeper01.local", port: "2181" }

```

## Dependencies

`Clickhouse` depends on `Zookeeper` to achieve **consistency**

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: clickhouse
      roles:
         - { role: javigs82.ansible_role_clickhouse, clickhouse_display_name: "e-commerce" }

## References

 - https://clickhouse.tech/docs/en/getting-started
 - https://docs.ansible.com/

## Author

* javigs82 [github](https://github.com/javigs82/)

### Acknowledgments

 - https://github.com/AlexeySetevoi/ansible-clickhouse
 - https://github.com/nl2go/ansible-role-clickhouse
 - https://github.com/idealista/clickhouse_role

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details
