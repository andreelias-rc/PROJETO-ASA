# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 512
    vb.cpus = 1
    vb.linked_clone = true
    vb.check_guest_additions = false
    vb.gui = false
  end

  # O DHCP da rede privada sera fornecido pelo servidor arq.
  config.trigger.before :"Vagrant::Action::Builtin::WaitForCommunicator", type: :action do |t|
    t.info = "Desabilitando o DHCP do VirtualBox na rede privada"
    t.run = {
      inline: "VBoxManage dhcpserver modify --interface=vboxnet0 --disable"
    }
  end

  # Configuracoes aplicadas a todas as maquinas.
  config.vm.provision "ansible" do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbooks/conf-comum.yml"
  end

  config.vm.define "arq" do |arq|
    arq.vm.hostname = "arq.andre.devops"
    arq.vm.network "private_network", ip: "192.168.56.124", name: "vboxnet0"

    (1..3).each do |i|
      arq.vm.disk :disk, size: "10GB", name: "arq-disco-#{i}"
    end

    arq.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/conf-arq.yml"
    end
  end

  config.vm.define "db" do |db|
    db.vm.hostname = "db.andre.devops"
    db.vm.network "private_network", type: "dhcp", mac: "0800273A0001", name: "vboxnet0"

    db.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/conf-db.yml"
    end
  end

  config.vm.define "app" do |app|
    app.vm.hostname = "app.andre.devops"
    app.vm.network "private_network", type: "dhcp", mac: "0800273A0002", name: "vboxnet0"

    app.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/conf-app.yml"
    end
  end

  config.vm.define "cli" do |cli|
    cli.vm.hostname = "cli.andre.devops"
    cli.vm.network "private_network", type: "dhcp", name: "vboxnet0"
    cli.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
    end

    cli.vm.provision "ansible" do |ansible|
      ansible.compatibility_mode = "2.0"
      ansible.playbook = "playbooks/conf-cli.yml"
    end
  end
end
