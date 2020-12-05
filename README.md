# Ansible Clickhouse

This role is in charge of configuring & installing a clickhouse cluster with N shard and M replicas in `centos7`.
It builds the cluster based in ansible inventory groups, so following groups are mandatory to run the cluster:

 - clickhouse: contains all clickhouse inventory hosts. These host must set their hostname like: `ch01-shard01-replica01`
 with regex like: `^ch\\d{2}-shard\\d{2}-replica\\d{2}`
 - zookeeper: contains all zookeeper inventory hosts. These hosts must set hostname like: `zookeeper01`

Take a look into [macros.xml](./templates/macros.xml.j2) to check how shard is being calculated in function of hostname: `clickhouse_shard_name`

**Note** that `{{ inventory_hostname }}` is the DNS or IP address while `{{ ansible_hostname }}` is the hostname of the machine.

In order to check if the hostname (`{{ ansible_hostname}}`) is properly set, check in the host machine with following command:

> hostname

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

## Architecture

**Clickhouse-cluster** is built on top of hostname, so ensure hostname is properly set for
being used in [defaults](./defaults/main.yml)

Cluster replica hostname must be defined as following:

> chX-shardY-replicaZ where regex is `^ch\\d{2}-shard\\d{2}-replica\\d{2}`

where **chX-shardY-replicaZ** is the hostname. Notice that this role implements vagrant-molecule and use special configuration to use service discovery. [Click here for more info](./molecule/default/prepare.yml)

**Notice** that the `OS hostname`  is not the same than `inventory_hostname`:
 - OS hostname: It is the name of the host. `hostname`
 - inventory_hostname: It is the URL (IP or DNS) of the host. URL must be resolved in order than cluster can communicate its nodes

Inventory hostname will be use to discover replicas and for enabling communications. Also, by its 

```INI
[clickhouse]
<URL-ch01-shard01-replica01> ansible_ssh=<ip>
<URL-ch01-shard01-replica02> ansible_ssh=<ip>
...
<URL-ch01-shard02-replica01> ansible_ssh=<ip>
<URL-ch01-shard02-replica02> ansible_ssh=<ip>
...
<URL-chX-shardY-replicaZ> ansible_ssh=<ip>

[zookeeper]
<URL-zookeeper01> ansible_ssh=<ip>
...
<URL-zookeeperN> ansible_ssh=<ip>

```
where URL is can be the IP or the DNS that will be resolved in runtime.

Notice that in order to build the cluster, `clickhouse` group and `zookeeper` group are mandatory

### Design

 - Download: From yandex rpm repository. Downgrade is allowed by `clickhouse_allow_downgrade` property.
 - Config: Ensure `clickhouse` group & user. Ensure paths and config files
 - Install: Download and install by yum
 - Users: Dynamic list to manage users. Password management is not implemented
 - RBAC: TO BE IMPLEMENTED


## Role Variables

Check variables in [defaults](./defaults/main.yml)

### Cluster Definition

Use these variable to set up the main definition. Notice that cluster configuration relies on hostname and with that, 
[macros.xml](./templates/macros.xml.j2) is dynamically built based on regex


```yml

clickhouse_version: "20.9.5.5"
clickhouse_allow_downgrade: false
clickhouse_display_name: "mycluster"
clickhouse_shard_name: "{{ ansible_hostname | regex_search('^ch\\d{2}-(shard\\d{2})-replica\\d{2}', '\\1') | first }}"

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
  merge_tree_config: []

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
