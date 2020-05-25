# -*- mode: ruby -*-
# vi: set ft=ruby :

$subnet = "192.168.33"
$num_instances = 3
$vm_memory_master = 1536
$vm_memory_slave = 1024
$vm_memory_livy = 768
$vm_cpus = 1
$instance_prefix = "spark"


Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"

  max_slaves = $num_instances - 1

  config.vm.provider "virtualbox" do |v|
    v.linked_clone = true
    v.cpus = $vm_cpus
  end

  config.vm.define "livy" do |livy|
    livy.vm.hostname = "livy"
    livy.vm.network "private_network", ip: "#{$subnet}.5"

    livy.vm.provider "virtualbox" do |v|
      v.memory = $vm_memory_livy
    end
  end

  config.vm.define "master" do |master|
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "#{$subnet}.10"

    master.vm.provider "virtualbox" do |v|
      v.memory = $vm_memory_master
    end

    master.trigger.after :destroy, :halt do |trigger|
      trigger.info = "Cleaning up everything..."
      trigger.run = {inline: "rm -rf target output"}
    end
  end

  (1..max_slaves).each do |i|
    config.vm.define "slave#{i}" do |slave|
      slave.vm.hostname = "slave#{i}"
      slave.vm.network "private_network", ip: "#{$subnet}.#{i+1}0"

      slave.vm.provider "virtualbox" do |v|
        v.memory = $vm_memory_slave
      end

      if i == max_slaves
        slave.vm.provision "ansible" do |ansible|
          ansible.playbook = "spark.yml"
          ansible.inventory_path = "inventory.yml"
          ansible.verbose = "v"
          ansible.limit = "all"
        end
      end
    end
  end
end