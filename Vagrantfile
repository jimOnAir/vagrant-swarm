# -*- mode: ruby -*-
# vi: set ft=ruby :
IMAGE_NAME = "ubuntu/focal64"
WORKERS_COUNT = 2
ADDITIONAL_MANAGERS_COUNT = 2
SUBNET_BASE = "192.168.200."
APT_PROXY_IP = SUBNET_BASE + "2"
FIRST_MANAGER_IP = SUBNET_BASE + "3"
DOMAIN_NAME = 'local'

#Ansible groups config
ANSIBLE_GROUPS = {
  "additional_managers" => ["manager-[2:#{ADDITIONAL_MANAGERS_COUNT + 1}]"],
  "apt-proxy-group" => ["apt-proxy"],
  "first_manager" => ["manager-1"],
  "managers" => ["manager-[1:#{ADDITIONAL_MANAGERS_COUNT + 1}]"],
  "workers" => ["worker-[1:#{WORKERS_COUNT}]"],
}

ANSIBLE_RAW_ARGS = [
  # '-vvvv',
  '--diff',
]
STACKS_PLACEMENT = {
  "portainer" => 'manager-1'
}

NODE_LABELS = '{[ {"name": "manager-1", "labels": {"portainer": "true"}} ]}'
TRAEFIK_AUTH_BASIC = '{ "users": [ {"username": "admin", "password": "admin" } ]}'
PORTAINER_ADMIN_PASSWORD = 'admin'
DOCKER_CREDENTIALS = '{ "name": "user", "password": "12345678" }'

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

  config.vm.provider "virtualbox" do |vm|
    vm.memory = "1024"
    vm.cpus = 1
  end

  config.vm.define "apt-proxy" do |apt_proxy|
    apt_proxy.vm.box = IMAGE_NAME
    apt_proxy.vm.network "private_network", ip: APT_PROXY_IP
    apt_proxy.vm.network "forwarded_port", guest: 3142, host: 3142, auto_correct: true
    apt_proxy.vm.hostname = "apt-proxy"
    apt_proxy.vm.provision "ansible" do |ansible|
      ansible.playbook = "swarm-setup/apt-proxy-playbook.yml"
      ansible.groups = ANSIBLE_GROUPS
      ansible.raw_arguments = ANSIBLE_RAW_ARGS
      ansible.extra_vars = {
        node_labels: NODE_LABELS
      }
    end
  end

  config.vm.define "manager-1" do |leader|
    leader.vm.box = IMAGE_NAME
    leader.vm.network "private_network", ip: FIRST_MANAGER_IP
    leader.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
    leader.vm.network "forwarded_port", guest: 443, host: 8443, auto_correct: true
    leader.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true
    leader.vm.network "forwarded_port", guest: 4000, host: 4000, auto_correct: true


    leader.vm.hostname = "manager-1"
    leader.vm.provision "ansible" do |ansible|
      ansible.playbook = "swarm-setup/first-manager-playbook.yml"
      ansible.groups = ANSIBLE_GROUPS
      ansible.raw_arguments = ANSIBLE_RAW_ARGS
      ansible.extra_vars = {
        apt_proxy: APT_PROXY_IP,
        docker_user: "vagrant",
        docker_credentials: DOCKER_CREDENTIALS,
        domain: DOMAIN_NAME,
        first_manager_ip: FIRST_MANAGER_IP,
        node_labels: NODE_LABELS,
        portainer_node: STACKS_PLACEMENT['portainer'],
        portainer_admin_password: PORTAINER_ADMIN_PASSWORD,
        traefik_access_list: TRAEFIK_AUTH_BASIC,
      }
    end
  end

  (1..ADDITIONAL_MANAGERS_COUNT).each do |index|
    config.vm.define "manager-#{index + 1}" do |manager|
      manager.vm.box = IMAGE_NAME
      manager.vm.network "private_network", ip: SUBNET_BASE + "1#{index + 10}"
      manager.vm.network "forwarded_port", guest: 80, host: "#{ 8080 + index*10 }", auto_correct: true
      manager.vm.network "forwarded_port", guest: 443, host: "#{ 8443 + index*10 }", auto_correct: true
      manager.vm.network "forwarded_port", guest: 3000, host: "#{ 3000 + index*10 }", auto_correct: true
      manager.vm.network "forwarded_port", guest: 4000, host: "#{ 4000 + index*10 }", auto_correct: true

      manager.vm.hostname = "manager-#{index + 1}"
      manager.vm.provision "ansible" do |ansible|
        ansible.playbook = "swarm-setup/swarm-node-playbook.yml"
        ansible.groups = ANSIBLE_GROUPS
        ansible.raw_arguments = ANSIBLE_RAW_ARGS
        ansible.extra_vars = {
          advertise_addr: SUBNET_BASE + "1#{index + 10}",
          docker_credentials: DOCKER_CREDENTIALS,
          apt_proxy: APT_PROXY_IP,
          docker_user: "vagrant",
          first_manager_ip: FIRST_MANAGER_IP,
          join_token: 'join-token-manager',
        }
      end
    end
  end

  (1..WORKERS_COUNT).each do |index|
    config.vm.define "worker-#{index}" do |worker|
      worker.vm.box = IMAGE_NAME
      worker.vm.network "private_network", ip: SUBNET_BASE + "#{index + 10}"
      worker.vm.network "forwarded_port", guest: 80, host: "#{ 8080 + index*10 }", auto_correct: true
      worker.vm.network "forwarded_port", guest: 443, host: "#{ 8443 + index*10 }", auto_correct: true
      worker.vm.network "forwarded_port", guest: 3000, host: "#{ 3000 + index*10 }", auto_correct: true
      worker.vm.hostname = "worker-#{index}"
      worker.vm.provision "ansible" do |ansible|
        ansible.playbook = "swarm-setup/swarm-node-playbook.yml"
        ansible.groups = ANSIBLE_GROUPS
        ansible.raw_arguments = ANSIBLE_RAW_ARGS
        ansible.extra_vars = {
          advertise_addr: SUBNET_BASE + "#{index + 10}",
          apt_proxy: APT_PROXY_IP,
          docker_credentials: DOCKER_CREDENTIALS,
          docker_user: "vagrant",
          first_manager_ip: FIRST_MANAGER_IP,
          join_token: 'join-token-worker',
        }
      end
    end
  end
end

puts [
"Redirect ports to host:80 and host:443",
"Add the following lines to /etc/hosts",
"127.0.0.1      portainer.#{DOMAIN_NAME}",
"127.0.0.1      traefik.#{DOMAIN_NAME}",
"",
"Add port redirection to services to be available on 80 and 443 ports:",
"sudo iptables -t nat -I OUTPUT -p tcp -d 127.0.0.1 --dport 80 -j REDIRECT --to-ports 8080",
"sudo iptables -t nat -I OUTPUT -p tcp -d 127.0.0.1 --dport 443 -j REDIRECT --to-ports 8443",
"",
"Remove port redirection:",
"sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 80 -j REDIRECT --to-ports 8080",
"sudo iptables -t nat -D OUTPUT -p tcp -d 127.0.0.1 --dport 443 -j REDIRECT --to-ports 8443",
"",
"Create docker context:",
"docker context create vagrant --docker \"host=ssh://vagrant@localhost:2222\"",
"",
"Add vagrant ssh key:",
"eval $(ssh-agent)",
"ssh-add ~/.vagrant.d/insecure_private_key",
"",
"Invoke docker command on remove host:",
"docker --context vagrant ps"
].join("\n")
