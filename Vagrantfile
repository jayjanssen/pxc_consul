# -*- mode: ruby -*-
# vi: set ft=ruby :

# Assumes a box from https://github.com/jayjanssen/packer-percona

# This sets up 3 nodes with a common PXC, but you need to run bootstrap.sh to connect them.

require File.dirname(__FILE__) + '/lib/vagrant-common.rb'

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'aws'
pxc_version = "56"

# Node group counts and aws security groups (if using aws provider)
pxc_nodes = 1
consul_nodes = 1
test_nodes = 1

# AWS configuration
aws_region = "us-east-1"
aws_ips='private' # Use 'public' for cross-region AWS.  'private' otherwise (or commented out)
# pxc_security_groups = ['default','pxc']
# consul_security_groups = ['default','consul']
# test_security_groups = ['default']
pxc_security_groups = nil
consul_security_groups = nil
test_security_groups = nil

# computed values
consul_join = Array.new( consul_nodes ){ |i| "consul" + (i+1).to_s }.join( ' ' )

# configuration passed to puppet on each node. 
global_config = {
  'default_interface' => 'eth0',
  
  # Consul configuration
  'enable_consul' => true,
  'datacenter' => aws_region,
  'join_cluster' => consul_join,

  # Sysbench common configuration
  'tables' => 200,
  'rows' => 1000000,
  'threads' => 32,
  # 'tx_rate' => 10 
}

Vagrant.configure("2") do |config|
  config.vm.box = "perconajayj/centos-x86_64"
  config.vm.box_version = "~> 7"
  config.ssh.username = "root"
  
  # Create all the consul servers first
  (1..consul_nodes).each do |i|
    name = "consul" + i.to_s
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.provision :hostmanager
      node_config.vm.network :private_network, type: "dhcp"

      
      # Forward Consul UI port
      node_config.vm.network "forwarded_port", guest: 8500, host: 8500 + i, protocol: 'tcp'
      
      # Provisioners
      provision_puppet( node_config, "consul_server.pp" ) { |puppet|  
        puppet.facter = global_config.merge({
          'bootstrap_expect' => consul_nodes,
          'node_name' => name,
        })
      }

      # Providers
      provider_vmware( name, node_config )
      provider_virtualbox( name, node_config ) { |vb, override|        
        # Override the bind_addr on vbox to use the backend network
        provision_puppet( override, "consul_server.pp" ) {|puppet|
          puppet.facter = {
            'default_interface' => 'eth1'
          }
        }
      }
      provider_aws( name, node_config, 'm3.medium', aws_region, consul_security_groups, aws_ips)
    end
  end
 
  # Test nodes
  (1..test_nodes).each do |i|
    name = "test" + i.to_s
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.network :private_network, type: "dhcp"
      node_config.vm.provision :hostmanager
            
      # Provisioners
      provision_puppet( node_config, "sysbench.pp" ) { |puppet|  
        puppet.facter = global_config.merge({
            # Consul setup
            'node_name' => name,             
            'join_cluster' => consul_join,
            'mysql_host' => 'node1.node.consul'             # sysbench setup
          })
      }

      # Providers
      provider_vmware( name, node_config )
      provider_virtualbox( name, node_config ) { |vb, override|
        # Override the bind_addr on vbox to use the backend network        

        provision_puppet( override, "sysbench.pp" ) {|puppet|
          puppet.facter = { 'default_interface' => 'eth1' }
        }
      }  
      provider_aws( name, node_config, 'c3.large', aws_region, test_security_groups, aws_ips)
    end
  end 

  # Create the PXC nodes
  (1..pxc_nodes).each do |i|
    name = "node" + i.to_s
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.provision :hostmanager
      node_config.vm.network :private_network, type: "dhcp"

      # Provisioners
      # provision_puppet( node_config, "mount.pp" ) { |puppet|
      #   puppet.facter = {
      #     'mount_point' => '/gcache',
      #     'mount_dev' => 'xvdm',
      #     'mount_dev_scheduler' => 'deadline',
      #     'mount_owner' => 'root',
      #     'mount_group' => 'root',
      #     'mount_mode' => '0777'
      #   }
      # }

      provision_puppet( node_config, "pxc_server.pp" ) { |puppet| 
        puppet.facter = global_config.merge({
          # Consul setup
          'node_name' => name,
          
          # Sysbench setup
          'sysbench_load' => (i == 1 ? true : false ),
          # 'sysbench_load' => false,
          'sysbench_skip_test_client' => true,
          
          # PCT setup
          'percona_agent_api_key' => ENV['PERCONA_AGENT_API_KEY'],
          'vividcortex_api_key' => ENV['VIVIDCORTEX_API_KEY'],

          # PXC setup
          'datadir_dev' => 'dm-2',  # on VMs anyway, override for AWS or other
          'datadir_dev_scheduler' => 'noop',
          "percona_server_version"  => pxc_version,
          'innodb_buffer_pool_size' => '24G',
          'innodb_log_file_size' => '4G',
          'innodb_flush_log_at_trx_commit' => '0',
          'wsrep_cluster_address' => 'gcomm://pxc.service.consul',
          'pxc_bootstrap_node' => (i == 1 ? true : false ),
          'wsrep_provider_options' => 'gcache.size=50G; gcs.fc_limit=1024',
          'wsrep_slave_threads' => 32,
          'extra_mysqld_config' => '
wsrep_sst_donor = node2,

[sst]
compressor="pigz"
decompressor="pigz -d"
inno-apply-opts="--use-memory=20G"
'
        })
      }

      # Providers
      provider_aws( name, node_config, 'c3.4xlarge', aws_region, pxc_security_groups, aws_ips) { |aws, override|
        # aws.block_device_mapping = [
        #   {
        #     'DeviceName' => "/dev/sdl",
        #     'VirtualName' => "mysql_data",
        #     'Ebs.VolumeSize' => 50,
        #     'Ebs.DeleteOnTermination' => true,
        #     'Ebs.VolumeType' => 'gp2'
        #   }
        #   # ,
        #   # {
        #   #   'DeviceName' => "/dev/sdm",
        #   #   'VirtualName' => "gcache",
        #   #   'Ebs.VolumeSize' => 20,
        #   #   'Ebs.DeleteOnTermination' => true
        #   # }
        # ]

        # provision_puppet( override, "pxc_server.pp" ) {|puppet| puppet.facter = { 'datadir_dev' => 'xvdl' }}

        aws.block_device_mapping = [
          { 'DeviceName' => "/dev/sdb", 'VirtualName' => "ephemeral0" },
          { 'DeviceName' => "/dev/sdc", 'VirtualName' => "ephemeral1" }
        ]
        provision_puppet( override, "pxc_server.pp" ) {|puppet| puppet.facter = { 
            'softraid' => true,
            'softraid_dev' => '/dev/md0',
            'softraid_level' => 'stripe',
            'softraid_devices' => '2',
            'softraid_dev_str' => '/dev/xvdb /dev/xvdc',

            'datadir_dev' => 'md0'
          }}
      }

    end
  end

end
