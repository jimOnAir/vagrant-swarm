# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'ipaddr'

IMAGE_NAME = "ubuntu/focal64"
DOMAIN_NAME = 'local'
WORKERS_COUNT = 2
ADDITIONAL_MANAGERS_COUNT = 2
ANSIBLE_RAW_ARGS = [
  # '-vvvv',
  '--diff',
]
STACKS_PLACEMENT = {
  "portainer" => 'manager-1',
  "elasticsearch" => 'manager-1',
  "jenkins" => 'manager-1',
}

NODE_LABELS = '{[ {"name": "manager-1", "labels": {"portainer": "true"}} ]}'
ELASTIC_PASSWORD = 'elastic'
TRAEFIK_AUTH_BASIC = '{ "users": [ {"username": "admin", "password": "admin" } ]}'
PORTAINER_ADMIN_PASSWORD = 'admin'

#Ansible groups config
ANSIBLE_GROUPS = {
  "additional_managers" => ["manager-[2:#{ADDITIONAL_MANAGERS_COUNT + 1}]"],
  "apt-proxy-group" => ["apt-proxy"],
  "first_manager" => ["manager-1"],
  "managers" => ["manager-[1:#{ADDITIONAL_MANAGERS_COUNT + 1}]"],
  "workers" => ["worker-[1:#{WORKERS_COUNT}]"],
}

def next_ip(previous_ipaddr)
  next_ipaddr = IPAddr.new(previous_ipaddr.to_i + 1, Socket::AF_INET)
end

SUBNET_IPADDR = IPAddr.new("192.168.200.0/24")
last_ipaddr = SUBNET_IPADDR
GATEWAY_IPADDR = next_ip(last_ipaddr)
last_ipaddr = GATEWAY_IPADDR
APT_PROXY_IPADDR = next_ip(last_ipaddr)
last_ipaddr = APT_PROXY_IPADDR
MANAGER_1_IPADDR = next_ip(last_ipaddr)
last_ipaddr = MANAGER_1_IPADDR

IP_LIST = {
  "apt-proxy" => APT_PROXY_IPADDR.to_s,
  "manager-1" => MANAGER_1_IPADDR.to_s,
}

(2..ADDITIONAL_MANAGERS_COUNT + 1).each do |index|
  manager_ip = next_ip(last_ipaddr)
  last_ipaddr = manager_ip
  IP_LIST["manager-#{index}"] = manager_ip.to_s
end

(1..WORKERS_COUNT).each do |index|
  worker_ip = next_ip(last_ipaddr)
  last_ipaddr = worker_ip
  IP_LIST["worker-#{index}"] = worker_ip.to_s
end

PROMETHEUS_HOSTS=[]
IP_LIST.each{|k,v| PROMETHEUS_HOSTS.append("#{k}:#{v}") if not k == 'apt-proxy'}

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

  config.vm.provider "virtualbox" do |vm|
    vm.memory = "2048"
    vm.cpus = 1
    vm.linked_clone = true
  end

  config.vm.define "apt-proxy" do |apt_proxy|
    apt_proxy.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "512"]
    end
    apt_proxy.vm.box = IMAGE_NAME
    apt_proxy.vm.network "private_network", ip: IP_LIST['apt-proxy']
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

  config.vm.define "manager-1" do |manager|
    manager.vm.box = IMAGE_NAME
    manager.vm.network "private_network", ip: IP_LIST['manager-1']
    manager.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
    manager.vm.network "forwarded_port", guest: 443, host: 8443, auto_correct: true
    manager.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "4096"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]

    end

    manager.vm.hostname = "manager-1"
    manager.vm.provision "ansible" do |ansible|
      ansible.playbook = "swarm-setup/first-manager-playbook.yml"
      ansible.groups = ANSIBLE_GROUPS
      ansible.raw_arguments = ANSIBLE_RAW_ARGS
      ansible.extra_vars = {
        apt_proxy: IP_LIST['apt-proxy'],
        docker_user: "vagrant",
        domain: DOMAIN_NAME,
        elasticsearch_node: STACKS_PLACEMENT['elasticsearch'],
        elasticsearch_password: ELASTIC_PASSWORD,
        first_manager_ip: IP_LIST['manager-1'],
        jenkins_node: STACKS_PLACEMENT['jenkins'],
        node_labels: NODE_LABELS,
        portainer_admin_password: PORTAINER_ADMIN_PASSWORD,
        portainer_node: STACKS_PLACEMENT['portainer'],
        prometheus_target_hosts: PROMETHEUS_HOSTS,
        traefik_access_list: TRAEFIK_AUTH_BASIC,
      }
    end
  end

  (2..ADDITIONAL_MANAGERS_COUNT + 1).each do |index|
    config.vm.define "manager-#{index}" do |manager|
      manager.vm.box = IMAGE_NAME
      manager.vm.network "private_network", ip: IP_LIST["manager-#{index}"]
      manager.vm.network "forwarded_port", guest: 80, host: "#{ 8080 + index*10 }", auto_correct: true
      manager.vm.network "forwarded_port", guest: 443, host: "#{ 8443 + index*10 }", auto_correct: true
      manager.vm.hostname = "manager-#{index}"
      manager.vm.provision "ansible" do |ansible|
        ansible.playbook = "swarm-setup/swarm-node-playbook.yml"
        ansible.groups = ANSIBLE_GROUPS
        ansible.raw_arguments = ANSIBLE_RAW_ARGS
        ansible.extra_vars = {
          advertise_addr: IP_LIST["manager-#{index}"],
          apt_proxy: IP_LIST['apt-proxy'],
          docker_user: "vagrant",
          first_manager_ip: IP_LIST['manager-1'],
          join_token: 'join-token-manager',
        }
      end
    end
  end

  (1..WORKERS_COUNT).each do |index|
    config.vm.define "worker-#{index}" do |worker|
      worker.vm.box = IMAGE_NAME
      worker.vm.network "private_network", ip: IP_LIST["worker-#{index}"]
      worker.vm.network "forwarded_port", guest: 80, host: "#{ 8080 + index*10 }", auto_correct: true
      worker.vm.network "forwarded_port", guest: 443, host: "#{ 8443 + index*10 }", auto_correct: true
      worker.vm.hostname = "worker-#{index}"
      worker.vm.provision "ansible" do |ansible|
        ansible.playbook = "swarm-setup/swarm-node-playbook.yml"
        ansible.groups = ANSIBLE_GROUPS
        ansible.raw_arguments = ANSIBLE_RAW_ARGS
        ansible.extra_vars = {
          advertise_addr: IP_LIST["worker-#{index}"],
          apt_proxy: IP_LIST['apt-proxy'],
          docker_user: "vagrant",
          first_manager_ip: IP_LIST['manager-1'],
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
