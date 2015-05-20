# -*- mode: ruby -*-
# vi: set ft=ruby :
# Vagrantfile to install 4-node Openstack Icehouse with puppet master
# Author: Philip Cheong

VAGRANTFILE_API_VERSION = "2"

# Explicitly define all puppet module version for openstack that we want
$install_puppet_modules = <<SCRIPT
/usr/local/bin/puppet module install puppetlabs-openstack  --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install dprince-qpid          --version 1.0.2 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install duritong-sysctl       --version 0.0.4 --force --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install garethr-erlang        --version 0.3.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-apache     --version 1.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-ceilometer --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-cinder     --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-glance     --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-heat       --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-horizon    --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-keystone   --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-rabbitmq   --version 3.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-mongodb    --version 0.8.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-mysql      --version 2.3.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-neutron    --version 4.2.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-nova       --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-rsync      --version 0.3.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-swift      --version 4.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-tempest    --version 3.0.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-vcsrepo    --version 0.2.0 --force --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-vswitch    --version 0.3.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-xinetd     --version 1.3.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install saz-memcached         --version 2.5.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install saz-ssh               --version 1.4.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install stahnma-epel          --version 0.1.0 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
/usr/local/bin/puppet module install puppetlabs-ntp        --version 3.0.3 --ignore-dependencies --modulepath '/etc/puppetlabs/puppet/modules'
# hackery needed to get the module version above to play nice
chmod u+w /etc/puppetlabs/puppet/modules/openstack/metadata.json
sed -i -e 's@duritong/sysctl\",\"version_requirement\":\"@duritong/sysctl\",\"version_requirement\":\"\>=@g' /etc/puppetlabs/puppet/modules/openstack/metadata.json
SCRIPT

$hiera_setup = <<SCRIPT
mkdir /etc/puppetlabs/puppet/hieradata
cp /etc/puppetlabs/puppet/modules/openstack/examples/common.yaml /etc/puppetlabs/puppet/hieradata/openstack.yaml
chgrp -R pe-puppet /etc/puppetlabs/puppet/hieradata
sed -i 's/^openstack::network::external::gateway:.*/openstack::network::external::gateway: 192.168.22.1/g' /etc/puppetlabs/puppet/hieradata/openstack.yaml
sed -i 's/^openstack::network::external::dns:.*/openstack::network::external::dns: 8.8.8.8/g' /etc/puppetlabs/puppet/hieradata/openstack.yaml
sed -i "s@^openstack::network::neutron::private:.*@openstack::network::neutron::private: \'10.10.10.0/24\'@g" /etc/puppetlabs/puppet/hieradata/openstack.yaml
sed -i "s@^openstack::nova::libvirt_type:.*@openstack::nova::libvirt_type: \'qemu\'@g" /etc/puppetlabs/puppet/hieradata/openstack.yaml
sed -i "s/^openstack::keystone::admin_email:.*/openstack::keystone::admin_email: \'philip.cheong@elastx.se\'/g" /etc/puppetlabs/puppet/hieradata/openstack.yaml
awk '{print} sub(/- global/,"- openstack")' /etc/puppetlabs/puppet/hiera.yaml > hiera.new && mv -f hiera.new /etc/puppetlabs/puppet/hiera.yaml
sed -i 's@:datadir:@:datadir: /etc/puppetlabs/puppet/hieradata@g' /etc/puppetlabs/puppet/hiera.yaml
service pe-puppet restart
SCRIPT

# Saved for posterity
#sed -i -e '/- global/{:a;n;/^$/!ba;i\ \ - openstack' -e '}' /etc/puppetlabs/puppet/hiera.yaml
#sed -i -e 's/- global/&\'$'\n''  - openstack/' /etc/puppetlabs/puppet/hiera.yaml

$add_node_classes = <<SCRIPT
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production nodeclass:add[openstack::role::compute,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production nodeclass:add[openstack::role::controller,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production nodeclass:add[openstack::role::network,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production nodeclass:add[openstack::role::storage,skip]
SCRIPT

$add_nodes = <<SCRIPT
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production node:add[control.localnet,,openstack::role::controller,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production node:add[compute.localnet,,openstack::role::compute,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production node:add[network.localnet,,openstack::role::network,skip]
/opt/puppet/bin/rake -f /opt/puppet/share/puppet-dashboard/Rakefile RAILS_ENV=production node:add[storage.localnet,,openstack::role::storage,skip]
SCRIPT

unless Vagrant.has_plugin?('oscar')
  # Thanks to benesch for this tip https://github.com/WhoopInc/vagrant-s3auth
  # Attempt to install the plugin if it's not present
  system('vagrant plugin install oscar') || exit!

  # Relaunch so the plugin is detected
  exit system('vagrant', *ARGV)
end

Vagrant.configure('2') do |config|
  config.vm.box                      = 'centos-64-x64-vbox4210-nocm'
  config.vm.box_url                  = 'http://puppet-vagrant-boxes.puppetlabs.com/centos-64-x64-vbox4210-nocm.box'
  config.pe_build.download_root      = 'https://s3.amazonaws.com/pe-builds/released'
  config.pe_build.version            = '3.2.3'
  config.pe_build.filename           = 'puppet-enterprise-3.2.3-el-6-x86_64.tar.gz'

  config.vm.provision 'shell', inline: 'yum update -y'
  # Only here to assist with initial provitioning of machines. After puppet agent run iptables is reenabled and started 
  config.vm.provision 'shell', inline: 'chkconfig iptables off'
  config.vm.provision 'shell', inline: 'service iptables stop'
  config.vm.provision :hosts
  # Fix this for RabbitMQ
  config.vm.provision 'shell', inline: 'sed -i "s/^127.0.1.1/127.0.0.1/g" /etc/hosts'

  # Puppet Master config
  config.vm.define 'master' do |master|
    master.vm.hostname = 'master.localnet'
    master.vm.network :private_network, ip: "192.168.11.3"
    master.vm.network :private_network, ip: "192.168.22.3"
    master.vm.network :private_network, ip: "172.16.33.3"
    master.vm.network :private_network, ip: "172.16.44.3"
    master.vm.provider :virtualbox do |vbox|
      vbox.customize ["modifyvm", :id, "--memory", "1024"]
    end

    master.vm.provision :pe_bootstrap do |provisioner|
      provisioner.role = :master
      provisioner.master = "master.localnet"
    end

    master.vm.provision "shell", inline: $install_puppet_modules
    master.vm.provision "shell", inline: $hiera_setup
    master.vm.provision "shell", inline: $add_node_classes
    master.vm.provision "shell", inline: $add_nodes

  end # master

  # Contoller node config
  config.vm.define 'control' do |control|
    control.vm.hostname = 'control.localnet'
    control.vm.network :private_network, ip: "192.168.11.4"
    control.vm.network :private_network, ip: "192.168.22.4"
    control.vm.network :private_network, ip: "172.16.33.4"
    control.vm.network :private_network, ip: "172.16.44.4"
    control.vm.provider :virtualbox do |vbox|
      vbox.customize ["modifyvm", :id, "--memory", "1280"]
    end

    control.vm.provision :pe_bootstrap do |provisioner|
      provisioner.role = :agent
      provisioner.master = "master.localnet"
    end
    
    control.vm.provision "shell", inline: "/usr/local/bin/puppet agent -tvd &"
  end # control

  # Storage node config
  config.vm.define 'storage' do |storage|
    storage.vm.hostname = 'storage.localnet'
    storage.vm.network :private_network, ip: "192.168.11.5"
    storage.vm.network :private_network, ip: "192.168.22.5"
    storage.vm.network :private_network, ip: "172.16.33.5"
    storage.vm.network :private_network, ip: "172.16.44.5"
    storage.vm.provider :virtualbox do |vbox|
      vbox.customize ["modifyvm", :id, "--memory", "512"]
    end

    storage.vm.provision :pe_bootstrap do |provisioner|
      provisioner.role = :agent
      provisioner.master = "master.localnet"
    end
    
    storage.vm.provision "shell", inline: "/usr/local/bin/puppet agent -tvd &"
  end # storage

  # Network node config
  config.vm.define 'network' do |network|
    network.vm.hostname = 'network.localnet'
    network.vm.network :private_network, ip: "192.168.11.6"
    network.vm.network :private_network, ip: "192.168.22.6"
    network.vm.network :private_network, ip: "172.16.33.6"
    network.vm.network :private_network, ip: "172.16.44.6"
    network.vm.provider :virtualbox do |vbox|
      vbox.customize ["modifyvm", :id, "--memory", "640"]
      vbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end

    network.vm.provision :pe_bootstrap do |provisioner|
      provisioner.role = :agent
      provisioner.master = "master.localnet"
    end
    
    network.vm.provision "shell", inline: "/usr/local/bin/puppet agent -tvd &"
  end # network

  # Compute node config
  config.vm.define 'compute' do |compute|
    compute.vm.hostname = 'compute.localnet'
    compute.vm.network :private_network, ip: "192.168.11.7"
    compute.vm.network :private_network, ip: "192.168.22.7"
    compute.vm.network :private_network, ip: "172.16.33.7"
    compute.vm.network :private_network, ip: "172.16.44.7"
    compute.vm.provider :virtualbox do |vbox|
      vbox.customize ["modifyvm", :id, "--memory", "1280"]
      vbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end

    compute.vm.provision :pe_bootstrap do |provisioner|
      provisioner.role = :agent
      provisioner.master = "master.localnet"
    end
    
    compute.vm.provision "shell", inline: "/usr/local/bin/puppet agent -tvd &"
  end # compute
end
