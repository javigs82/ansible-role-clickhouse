# Clickhouse Cluster

This role install and configure clickhouse cluster for N shard and M replicas in `centos7`.

## Requirements

Please install following package to use `molecule` as TDD tool

```sh

python3 -m pip install --user "molecule[vagrant,lint]"
pip3 install testinfra


```

where `vagrant` is the driver and `testinfra` is the testing provider.

Assuming vagrant does not provide any dns solution, the following software is installed in prepare.yml

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

## Design

 - Download: From yandex rpm repository. Downgrade is allowed by `clickhouse_allow_downgrade` property.
 - Config: Ensure `clickhouse` group & user. Ensure paths and config files
 - Install: Download and install by yum
 - Users: Dynamic list to manage users. Password are not implemented
 - RBAC: TO BE IMPLEMENTED

### Cluster configuration

In order to "mount" the cluster, `hostname` takes an important consideration.

Cluster is defined in a yaml structure like:

> shardN.replicaN.local

where **shardN.replicaN** is the hostname and **local** is the domain to be resolved. Notice that this role implements vagrant-molecule and use special configuration to use service discovery. [Click here for more info](./molecule/default/prepare.yml)

## Role Variables

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.


## Dependencies

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

## TODO
 - macros: implement shard name if exists in replicas

## Author

* javigs82 [github](https://github.com/javigs82/)

## Acknowledges

 - https://github.com/AlexeySetevoi/ansible-clickhouse
 - https://github.com/nl2go/ansible-role-clickhouse
 - https://github.com/idealista/clickhouse_role

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details
