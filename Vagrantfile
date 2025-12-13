# -*- mode: ruby -*-
# vi: set ft=ruby :

nome1 = "caio"
nome2 = "gabriel"
xx = 22
yy = 32

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  config.ssh.insert_key = false

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider :virtualbox do |v|
    v.gui = false
    v.memory = 512
    v.linked_clone = true
    v.check_guest_additions = false
  end

  config.trigger.before :"Vagrant::Action::Builtin::WaitForCommunicator", type: :action do |t|
    t.warn = "Interrompe o servidor dhcp do virtualbox"
    t.run = {inline: "VBoxManage dhcpserver stop --interface vboxnet0"}
  end

  config.vm.provision "ansible" do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbooks/main.yml"
  end

  # Servidor de arquivos
  config.vm.define "arq" do |arq|
    arq.vm.hostname = "arq.#{nome1}.#{nome2}.devops"
    arq.vm.network :private_network, ip: "192.168.56.1#{xx}"
    (0..2).each do |x|
      arq.vm.disk :disk, size: "10GB", name: "disk-#{x}"
    end
  end

  # Servidor de banco de dados
  config.vm.define "db" do |db|
    db.vm.hostname = "db.#{nome1}.#{nome2}.devops"
    db.vm.network :private_network, mac: "0800273A0001", type: "dhcp"
  end

  # Servidor de aplicação
  config.vm.define "app" do |app|
    app.vm.hostname = "app.#{nome1}.#{nome2}.devops"
    app.vm.network :private_network, mac: "0800273A0002", type: "dhcp"
  end

  # Host Cliente
  config.vm.define "cli" do |cli|
    cli.vm.hostname = "cli.#{nome1}.#{nome2}.devops"
    cli.vm.network :private_network, type: "dhcp"

    cli.vm.provider :virtualbox do |v|
      v.memory = 1024
    end
  end
end
