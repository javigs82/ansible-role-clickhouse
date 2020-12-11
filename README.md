# Ansible Clickhouse

This role is in charge of configuring & installing a clickhouse cluster with N shard and M replicas. 
At this moment only `centos7` is implemented.

The cluster is built based on ansible inventory groups aka **inventory patterns**, so following groups are mandatory to run the cluster:

 - **clickhouse**: contains all clickhouse inventory hosts. These hosts must set their hostname like: `ch01-shard01-replica01`
 with regex like: `^ch\\d{2}-shard\\d{2}-replica\\d{2}`
 - **zookeeper**: contains all zookeeper inventory hosts.

Take a look into [defaults](./defaults/main.yml) to check how shard and replicas is being calculated in function of hostnames: 

**Note** that `{{ inventory_hostname }}` is the DNS or IP address while `{{ ansible_hostname }}` is the hostname of the machine.

In order to check if the hostname (`{{ ansible_hostname}}`) is properly set, check in the host machine with following command:

> hostname

## Requirements

* Vagrant with virtual box (see https://www.vagrantup.com/intro)
* ansible => 2.10
* python3
* pip3
* Molecule (see https://molecule.readthedocs.io/en/latest/installation.html) 
* Testinfra (see https://testinfra.readthedocs.io/en/latest/)


In order to install `molecule` use python3.

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

**Clickhouse-cluster** is built on top of hostname, so ensure hostname is properly set like

> `^ch\\d{2}-shard\\d{2}-replica\\d{2}`

where **chX-shardY-replicaZ** is the key providing a way to easily discovering which replica belongs what shard based on regex. 

**Notice** that the `ansible_hostname`  is not the same than `inventory_hostname`:
 - **ansible_hostname**: It is the name in the OS: do `hostname` in the host machine
 - **inventory_hostname**: It is the URL (IP or DNS) of the host and must be resolved by other cluster replicas.
 IP address or DN are both valid. 
 
`{{ ansible_hostname }}` will be used for discovering replicas while `{{ inventory_hostname}}` for enabling communications. 

An example of inventory could be:

```INI
[clickhouse]
<URL-ch01-shard01-replica01> ansible_host=<ip>
<URL-ch01-shard01-replica02> ansible_host=<ip>
...
<URL-ch01-shard02-replica01> ansible_host=<ip>
<URL-ch01-shard02-replica02> ansible_host=<ip>
...
<URL-chX-shardY-replicaZ> ansible_host=<ip>

[zookeeper]
<URL-zookeeper01> ansible_host=<ip>
...
<URL-zookeeperN> ansible_host=<ip>

```
where URL (inventory_hostname) can be the IP or the DNS that will be resolved in runtime.

Notice that in order to build the cluster, `clickhouse` group and `zookeeper` group are mandatory

### Design

 - Download: From yandex rpm repository. Downgrade is allowed by `clickhouse_allow_downgrade` property.
 - Config: Ensure `clickhouse` group & user. Ensure paths and config files. Discover replicas and shards based on regex.
 - Install: Download and install by yum
 - Users: Dynamic list to manage users. Password management is not implemented
 - RBAC: TO BE IMPLEMENTED


## Role Default Variables

Check variables in [defaults](./defaults/main.yml)

### Cluster Definition

Use these variable to set up the main definition. 
Notice that cluster configuration relies on hostname and with that, `clickhouse_replica_name` and `clickhouse_shard_name`
take relevance while `clickhouse_hostname_regex` is the regex of the hostname definition: `^ch\\d{2}-(shard\\d{2})-replica\\d{2}` See [vars](./vars/defaults.yml) for more info.

```yml

# clickhouse cluster definition
clickhouse_version: "20.9.5.5"
clickhouse_allow_downgrade: false
clickhouse_cluster_name: "mycluster"
# replica list are all host of a group
clickhouse_replica_list: "{{ groups['clickhouse'] }}"

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

These variable are related to networking configuration

```yml

clickhouse_http_port: 8123
clickhouse_https_port: 8443
clickhouse_tcp_port: 9000
clickhouse_tcp_secure_port: 9440
clickhouse_interserver_http: 9009

# see vars for clickhouse_listen_host_default
clickhouse_listen_host: "{{ clickhouse_listen_host_default + clickhouse_listen_host_custom }}"

```

**Notice** that `clickhouse_listen_host` must allow ch members listening

### Users

Use these vars to customize users. In order to remove a user use clickhouse config attributes:
https://clickhouse.tech/docs/en/operations/configuration-files/

```yml

clickhouse_users_list:
  - { user_name: "reader",
      profile: "readonly",
      password: "r3ad3R",
      networks: ["::/0"],
      quota: "default" }
  - { user_name: "javito",
      profile: "default",
      password: "javito",
      networks: ["::/0"],
      quota: "default" }
  - { user_name: "default",
      profile: "default",
      networks: ["::/0"],
      quota: "default" }
  - { user_name: "luis",
      profile: "default",
      password: "luis",
      networks: ["::1", "127.0.0.1"],
      quota: "default" }

```

### Zookeeper

Zookeeper host list is based on inventory groups pattern

```yml

# zookeeper is not at all mandatory. If zookeeper is not installed, replication must be accomplished by the client side
clickhouse_zookeeper_list: "{{ groups['zookeeper'] }}"
clickhouse_zookeeper_port: "2181"

```

## Role Vars Variables

They are variables that are greatest than defaults and inventory groups vars. They can only be overridden by some higher precedence, but normally they are not.

Notice that cluster configuration relies on hostname and with that, `clickhouse_replica_name` and `clickhouse_shard_name`
take relevance while `clickhouse_hostname_regex` is the regex of the hostname definition: `^ch\\d{2}-(shard\\d{2})-replica\\d{2}` See [vars](./vars/defaults.yml) for more info.

Check variables in [vars](./vars/main.yml)

```yml

# replica name is the inventory hostname and must be resolved by IP or DN
clickhouse_replica_name: "{{ inventory_hostname }}"

# sharding is based on hostname, so this regex obtains the proper shard of a hostname given
clickhouse_hostname_regex: "^ch\\d{2}-(shard\\d{2})-replica\\d{2}"
clickhouse_shard_name: "{{ ansible_hostname | regex_search(clickhouse_hostname_regex, '\\1') | first }}"
# shard list is calculated by replica list which must accomplish with regex: clickhouse_hostname_regex
clickhouse_shard_list: "{{ clickhouse_replica_list | map('extract', hostvars, 'ansible_hostname') | map('regex_search', clickhouse_hostname_regex, '\\1') | unique | map ('first') }}"

clickhouse_listen_host_default:
  - "{{ clickhouse_replica_name }}"
  - "127.0.0.1"
  - "::1"

```

They should NEVER be overridden as they are the core of how the rol discover and relate replicas with shards.

## Role Tags

Following tags are supported in this role:

 - `ch:configure`: For executing only config tasks.
 - `ch:install`: For download and install the software.

## Dependencies

`Clickhouse` depends on `Zookeeper` to achieve **consistency**

## Example Playbook

Including an example of how to use the rol

``` yml

    - hosts: my_clickhouse_group
      tasks:
        - include_role:
            name: javigs82.clickhouse
          vars:
            clickhouse_cluster_name: "e-commerce"
            clickhouse_replica_list: "{{ groups['my_clickhouse_group'] }}"
            clickhouse_zookeeper_list: "{{ groups['my_zookeeper_group'] }}"

```

## References

 - https://clickhouse.tech/docs/en/getting-started
 - https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html

## Author

* javigs82 [github](https://github.com/javigs82/)

### Acknowledgments

 - https://github.com/AlexeySetevoi/ansible-clickhouse
 - https://github.com/nl2go/ansible-role-clickhouse
 - https://github.com/idealista/clickhouse_role

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details
