# Vagrant Docker Swarm cluster setup

Automatic Docker Swarm cluster setup with Vagrant and Ansible

## Running

Just run:

```console
$ vagrant up
```

and wait for setup to complete

## Hosts

- `apt-proxy` - utility host with apt-cacher-ng to reduce network traffic
- `manager-1` - first Docker Swarm manager
- `manager-2`, etc...  - additional Docker Swarm managers
- `worker-`, etc... - Docker Swarm workers

## Configuration

Vagrant file contains following constants:

- `IMAGE_NAME = "ubuntu/focal64"` - Vagrant box image. `"ubuntu/focal64"` - the only one is supported
- `WORKERS_COUNT = 2` - Workers count
- `ADDITIONAL_MANAGERS_COUNT = 2` - Additional Docker Swarm managers count
- `SUBNET_BASE = "192.168.200."` - Private subnet address
- `APT_PROXY_IP = SUBNET_BASE + "2"` - `apt-proxy` ip address
- `FIRST_MANAGER_IP = SUBNET_BASE + "3"` - `manager-` ip address
- `DOMAIN = 'local'` - domain

## Ansible roles

The following roles are used:

- `apt-proxy` - setup apt-cacher-ng with docker repo proxy
- `apt-setup` - Vagrant related. Disable unattended updates
- `docker-install` - Install and configure Docker CE
- `swarm-first-manager` - Initialize Docker Swarm cluster and copy join tokens to ansible host
- `swarm-manager` - join manager to previously configured cluster using join token
- `swarm-worker` - join worker to previously configured cluster using join token

## Ansible playbooks

- `apt-proxy-playbook.yml` - setup `apt-proxy`
- `first-manager-playbook.yml` - setup `manager-1`
- `manager-playbook.yml` - setup `manager-2`, etc...
- `worker-playbook.yml` - setup `worker-`, etc...
