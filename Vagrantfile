# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :backup => {
        :box_name => "centos7",
        :ip_addr => '192.168.11.160',
    :disks => {
        :sata1 => {
            :dfile => home + '/yarkozloff/backup.vdi',
            :size => 2048,
            :port => 1
        }
    }
  },
  :client => {
        :box_name => "centos7",
        :ip_addr => '192.168.11.150',
    :disks => {
        :sata2 => {
            :dfile => home + '/yarkozloff/client.vdi',
            :size => 2048,
            :port => 2
        }
    }
  },
}

Vagrant.configure("2") do |config|

    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "256"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
            config.vm.provision "ansible" do |ansible|
              ansible.playbook = "playbook.yml"
              end 
        end
    end
  end