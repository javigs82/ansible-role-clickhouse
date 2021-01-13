*******
Molecule installation guide
*******

Requirements
============
* Vagrant with virtual box (see https://www.vagrantup.com/intro)
* ansible => 2.10
* python3
* pip3
* Molecule (see https://molecule.readthedocs.io/en/latest/installation.html) 

Dependencies
============

This roles assumes dns or hostname resolution, so DNS mechanism is installed (`nss-mdns` & `avahi-daemon`) 
during preparing phase [prepare.yml](./prepare.yml)

In order to install `Zookeeper`, this scenario uses ansible galaxy 

Install
=======

* python3 -m pip install --user "molecule"
* python3 -m pip install --user "molecule-vagrant"
