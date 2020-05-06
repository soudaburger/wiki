# Install Openstack Ansible AIO
<!-- TOC depthFrom:3 -->
- [Install Openstack Ansible AIO](#install-openstack-ansible-aio)
- [Assumptions](#assumptions)
- [Known Issues](#known-issues)
- [Install and Bootstrap openstack ansible](#install-and-bootstrap-openstack-ansible)


---
# Assumptions
    * This has ONLY been tested on Ubuntu 18.04 systems.

    * You have sufficient disk space to run scripts (80GB+).

    * All commands are run as ROOT.
---

# Known Issues
```bash
# fatal: [localhost]: FAILED! => {"msg": "'dict object' has no attribute 'interface'"}
export BOOTSTRAP_OPTS="bootstrap_host_public_address=<the address usable for access from outside the box>"
```


# Install and Bootstrap openstack ansible
```bash
# Clone git repository
git clone https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible

# Change to git root
cd /opt/openstack-ansible

# Export bootstrap options
export BOOTSTRAP_OPTS="bootstrap_host_public_address=192.168.1.2"

# Run ansible bootstrap
scripts/bootstrap-ansible.sh

# Run AIO bootstrap
scripts/bootstrap-aio.sh

# Change to playbooks directory
cd playbooks

# Run Host Setup
openstack-ansible setup-hosts.yaml

# Run Infrastructure Setup
openstack-ansible setup-infrastructure.yaml

# Run Openstack Setup
openstack-ansible setup-openstack.yaml
```
