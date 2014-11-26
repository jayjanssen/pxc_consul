# -*- mode: ruby -*-
# vi: set ft=ruby :

# Assumes a box from https://github.com/jayjanssen/packer-percona

# This sets up 3 nodes with a common PXC, but you need to run bootstrap.sh to connect them.

require File.dirname(__FILE__) + '/lib/vagrant-common.rb'

pxc_version = "56"

# Node group counts and aws security groups (if using aws provider)
pxc_nodes = 3
consul_nodes = 1
test_nodes = 1

# AWS configuration
aws_region = "us-west-1"
aws_ips='private' # Use 'public' for cross-region AWS.  'private' otherwise (or commented out)
pxc_security_groups = ['default','pxc']
consul_security_groups = ['default','consul']
test_security_groups = ['default']

# computed values
consul_join_array = Array.new( consul_nodes ){ |i| "consul" + (i+1).to_s }

Vagrant.configure("2") do |config|
  config.vm.box = "perconajayj/centos-x86_64"
  config.vm.box_version = "~> 7.0"
  config.ssh.username = "root"
  
  # Create all the consul servers first
  (1..consul_nodes).each do |i|
    name = "consul" + i.to_s
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.network :private_network, type: "dhcp"
      node_config.vm.provision :hostmanager
      
      # Forward Consul UI port
      node_config.vm.network "forwarded_port", guest: 8500, host: 8500 + i, protocol: 'tcp'
      
      # Provisioners
      provision_puppet( node_config, "consul_server.pp" ) { |puppet|  
          puppet.facter = {
            'join_cluster'   => consul_join_array,
            'bootstrap_expect' => consul_nodes,
            'node_name' => name,
            'datacenter' => aws_region
          }
      }

      # Providers
      provider_virtualbox( name, node_config, 256 ) { |vb, override|
        # Override the bind_addr on vbox to use the backend network
        provision_puppet( override, "consul_server.pp" ) {|puppet|
          puppet.facter = {
            'default_interface' => 'eth1'
          }
        }
      }  
      
      provider_aws( "#{name}", node_config, 'm1.small', aws_region, consul_security_groups, aws_ips)
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
          puppet.facter = {
            # Consul setup
            'enable_consul' => true,
            'node_name' => name,
            'join_cluster'   => consul_join_array,
            'datacenter' => aws_region,
            
            # sysbench setup
            'tables' => 1,
            'rows' => 1000000,
            'threads' => 32 * pxc_nodes,
            'mysql_host' => 'pxc.service.consul'
          }
      }

      # Providers
      provider_virtualbox( name, node_config, 256 ) { |vb, override|
        # Override the bind_addr on vbox to use the backend network
        provision_puppet( override, "sysbench.pp" ) {|puppet|
          puppet.facter = { 'default_interface' => 'eth1' }
        }
      }  
      
      provider_aws( "#{name}", node_config, 'c3.large', aws_region, test_security_groups, aws_ips)
    end
  end 

  # Create the PXC nodes
  (1..pxc_nodes).each do |i|
    name = "node" + i.to_s
    config.vm.define name do |node_config|
      node_config.vm.hostname = name
      node_config.vm.network :private_network, type: "dhcp"
      node_config.vm.provision :hostmanager
      
      # Provisioners
      provision_puppet( node_config, "pxc_server.pp" ) { |puppet| 
        puppet.facter = {
          # Consul setup
          'enable_consul' => true,
          'node_name' => name,
          'join_cluster'   => consul_join_array,
          'datacenter' => aws_region,
          
          # PXC setup
          "percona_server_version"  => pxc_version,
          'innodb_buffer_pool_size' => '1G',
          'innodb_log_file_size' => '1G',
          'innodb_flush_log_at_trx_commit' => '0',
          'pxc_bootstrap_node' => (i == 1 ? true : false ),
          'wsrep_cluster_address' => 'gcomm://pxc.service.consul',
          'wsrep_provider_options' => 'gcache.size=2G; gcs.fc_limit=1024',
          
          # Sysbench setup
          'sysbench_load' => (i == 1 ? true : false ),
          'sysbench_skip_test_client' => true,
          'tables' => 1,
          'rows' => 1000000,
          'threads' => 1,
          'tx_rate' => 10,
          
          # PCT setup
          'percona_agent_api_key' => ENV['PERCONA_AGENT_API_KEY']
        }
      }

      # Providers
      provider_virtualbox( name, node_config, 2048 ) { |vb, override|
        provision_puppet( override, "pxc_server.pp" ) {|puppet|
          puppet.facter = {
            'default_interface' => 'eth1',
            
            # PXC Setup
            'datadir_dev' => 'dm-2',
          }
        }
      }
  
      provider_aws( "PXC #{name}", node_config, 'm3.large', aws_region, pxc_security_groups, aws_ips) { |aws, override|
        aws.block_device_mapping = [
          { 'DeviceName' => "/dev/sdb", 'VirtualName' => "ephemeral0" }
        ]
        provision_puppet( override, "pxc_server.pp" ) {|puppet| puppet.facter = { 'datadir_dev' => 'xvdb' }}
      }

    end
  end

end

