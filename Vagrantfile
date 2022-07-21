# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 2
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.skip_tags = "ssl_cert"
    ansible.extra_vars = {
      this_in_vagrant: true
    }
  end
end
