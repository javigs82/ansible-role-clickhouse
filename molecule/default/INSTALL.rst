*******
Molecule installation guide
*******

Requirements
============
* ansible => 2.7
* python3
* pip3
* Molecule (see https://molecule.readthedocs.io/en/latest/installation.html) 
* Testinfra (see https://testinfra.readthedocs.io/en/latest/)

**Note** that molecule & testinfra have some issues over python 2.

Install
=======
```sh

pip3 install molecule[docker]
pip3 install testinfra
pip3 install debops
pip3 install netaddr
pip3 install future

```
