# -*- mode: ruby -*-
# vi: set ft=ruby :

# Define virtualbox boxes
boxes = [
    {
        :name => "node1",
        :eth1 => "192.168.50.101",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    },
    {
        :name => "node2",
        :eth1 => "192.168.50.102",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    },
    {
        :name => "node3",
        :eth1 => "192.168.50.103",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    }
]

# Lets check what kind of SSH key you have generated and upload it on vm
rsa_key = File.expand_path('~') + "/.ssh/id_rsa.pub"
dsa_key = File.expand_path('~') + "/.ssh/id_dsa.pub"

if FileTest.exists?(rsa_key)
  key = rsa_key
elsif  FileTest.exists?(dsa_key)
  key = dsa_key
end

Vagrant.configure(2) do |config|

  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.box = opts[:os]

      #config.ssh.insert_key = false
      ssh_public_key = File.read("#{key}")

      config.vm.network "private_network", ip: opts[:eth1]
      config.vm.hostname = opts[:name]
      config.vm.provider "virtualbox" do |v|
        v.memory = opts[:mem]
        v.cpus = opts[:cpu]
        v.name = opts[:name]
      end

      config.vm.provision "shell", inline: <<-SHELL
        touch /root/.ssh/authorized_keys
        echo "#{ssh_public_key}" >> /root/.ssh/authorized_keys
        chmod 640 /root/.ssh/authorized_keys
      SHELL

      #config.vm.provision :ansible do |ansible|
      #    ansible.playbook = "pre_provision.yml"
      #    ansible.inventory_path = "development/hosts"
      #    ansible.verbose = "v"
      #end
    end
  end
end
