# -*- mode: ruby -*-
# vi: set ft=ruby :

# Assumes a box from https://github.com/jayjanssen/packer-percona

# This sets up 3 nodes with a common PXC, but you need to run bootstrap.sh to connect them.

require File.dirname(__FILE__) + '/lib/vagrant-common.rb'

mysql_version = "56"

# Node names and ips (for local VMs)
# (Amazon) aws_region is where to bring up the node
# (Amazon) Security groups are 'default' (22 open) and 'pxc' (3306, 4567-4568,4444 open) for each respective region
# Don't worry about amazon config if you are not using that provider.
pxc_nodes = {
  'node1' => {
    'local_vm_ip' => '192.168.70.2',
    'aws_region' => 'us-east-1',
    'security_groups' => ['default','pxc']
  },
  'node2' => {
    'local_vm_ip' => '192.168.70.3',
    'aws_region' => 'us-east-1',
    'security_groups' => ['default','pxc'] 
  },
  'node3' => {
    'local_vm_ip' => '192.168.70.4',
    'aws_region' => 'us-east-1',
    'security_groups' => ['default','pxc']
  }
}


consul = {
  'consul1' => {
    'local_vm_ip' => '192.168.70.100',
    'aws_region' => 'us-east-1',
    'server_id' => 1,
    'security_groups' => ['default','consul']
  }
}

# Use 'public' for cross-region AWS.  'private' otherwise (or commented out)
aws_ips='private'


Vagrant.configure("2") do |config|
  config.vm.box = "perconajayj/centos-x86_64"
  config.vm.box_version = "~> 7.0"
  config.ssh.username = "root"
  
  config.hostmanager.enabled = true # Disable for AWS
  config.hostmanager.include_offline = true
  
  # Create all the consul servers first
  consul.each_pair { |name, params|
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.network :private_network, ip: params['local_vm_ip']
      node_config.vm.provision :hostmanager
      
      # Forward Consul UI port
      node_config.vm.network "forwarded_port", guest: 8500, host: 8500 + params['server_id'], protocol: 'tcp'
      
      # Provisioners
      provision_puppet( node_config, "consul_server.pp" ) { |puppet|  
          puppet.facter = {
            'join_cluster'   => consul.keys.join( ' ' ),
            'bootstrap_expect' => consul.length,
            'node_name' => name,
            'datacenter' => params['aws_region']
          }
      }

      # Providers
      provider_virtualbox( name, node_config, 256 ) { |vb, override|
        # Override the bind_addr on vbox to use the backend network
        provision_puppet( override, "consul_server.pp" ) {|puppet|
          puppet.facter = {
            'bind_addr' => params['local_vm_ip']
          }
        }
      }  
      
      provider_aws( "consul #{name}", node_config, 'm1.small', params['aws_region'], params['security_groups'], aws_ips)
    end
  }
  
  # Create the PXC nodes
  pxc_nodes.each_pair { |name, params|
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.network :private_network, ip: params['local_vm_ip']
      node_config.vm.provision :hostmanager
      
      # Provisioners
      provision_puppet( node_config, "pxc_server.pp" ) { |puppet| 
        puppet.facter = {
          # Consul setup
          'enable_consul' => true,
          'node_name' => name,
          'join_cluster'   => consul.keys.join( ' ' ),
          'datacenter' => params['aws_region'],
          
          # PXC setup
          "percona_server_version"  => mysql_version,
          'innodb_buffer_pool_size' => '1G',
          'innodb_flush_log_at_trx_commit' => '0',
          'pxc_bootstrap_node' => (name == 'node1' ? true : false ),
          'wsrep_cluster_address' => 'gcomm://' + pxc_nodes.keys.join( ',' ),
          # 'wsrep_provider_options' => 'gcache.size=2G; gcs.fc_limit=1024',
          
          # Sysbench setup
          'sysbench_load' => (name == 'node1' ? true : false ),
          'tables' => 1,
          'rows' => 1000000,
          'threads' => 1,
          
          # PCT setup
          'percona_agent_enabled' => true,
          'percona_agent_api_key' => '5b9ea296ca8a848f94ac31527970c783'
        }
      }

      # Providers
      provider_virtualbox( name, node_config, 2048 ) { |vb, override|
        provision_puppet( override, "pxc_server.pp" ) {|puppet|
          puppet.facter = {
            # Consul setup
            'bind_addr' => params['local_vm_ip'],
            
            # PXC Setup
            'datadir_dev' => 'dm-2',
            'extra_mysqld_config' => "wsrep_node_address = #{params['local_vm_ip']}"
          }
        }
      }
  
      provider_aws( "PXC #{name}", node_config, 'm3.2xlarge', params['aws_region'], params['security_groups'], aws_ips) { |aws, override|
        aws.block_device_mapping = [
          {
            'DeviceName' => "/dev/sdb",
            'VirtualName' => "ephemeral0"
          }
        ]
        provision_puppet( override, "pxc_server.pp" ) {|puppet| puppet.facter = { 'datadir_dev' => 'xvdb' }}
      }

    end
  }

end

