# Vagrant Docker Swarm cluster setup

Automatic Docker Swarm cluster setup with Vagrant and Ansible

## Prerequirements

The following software should be installed:

- [Vagrant](https://www.vagrantup.com/downloads)
- Vagrant Virtualbox plugin:

```console
vagrant plugin install virtualbox
```

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Python cryptography library](https://pypi.org/project/cryptography/)

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
- `deploy-portainer` - Deploy portainer stack on cluster
- `deploy-traefik` - Deploy traefik reverse-proxy stack
- `docker-install` - Install and configure Docker CE
- `swarm-first-manager` - Initialize Docker Swarm cluster and copy join tokens to ansible host
- `swarm-join-node` - join swarm node to previously configured cluster using join token

## Ansible playbooks

- `apt-proxy-playbook.yml` - setup `apt-proxy`
- `first-manager-playbook.yml` - setup `manager-1`
- `swarm-node-playbook.yml` - setup swarm nodes

## Aliases

Redirect ports to host:80 and host:443
Add the following lines to /etc/hosts

```
127.0.0.1      portainer.<DOMAIN_NAME>
127.0.0.1      traefik.<DOMAIN_NAME>
```

## Port redirection

Add port redirection to services to be available on 80 and 443 ports:

```console
sudo iptables -t nat -I OUTPUT -p tcp -d 127.0.0.1 --dport 80 -j REDIRECT --to-ports 8080
sudo iptables -t nat -I OUTPUT -p tcp -d 127.0.0.1 --dport 443 -j REDIRECT --to-ports 8443
```

Remove port redirection:

```console
sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 80 -j REDIRECT --to-ports 8080
sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 443 -j REDIRECT --to-ports 8443
```

## Work with remote docker

Create docker context:

``` console
$ docker context create vagrant --docker host=ssh://vagrant@localhost:2222
```

Add vagrant ssh key:

``` console
$ eval $(ssh-agent)
$ ssh-add ~/.vagrant.d/insecure_private_key
```

Invoke docker command on remove host

```console
docker --context vagrant ps
```
