# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'vagrant-openstack-provider'
require 'vagrant-bolt'

Vagrant.configure("2") do |config|
  #config.vm.box = "dummy"
  config.ssh.private_key_path = "~/.ssh/id_rsa-acceptance"
  config.ssh.insert_key = false

  config.vm.provider :openstack do |os, override|
    os.openstack_auth_url = ENV['OS_AUTH_URL']
    os.username           = ENV['OS_USERNAME']
    os.password           = ENV['OS_PASSWORD']
    os.project_name       = ENV['OS_PROJECT_NAME']
    os.domain_name        = ENV['OS_PROJECT_DOMAIN_ID']
    os.identity_api_version = '3'
    os.floating_ip_pool   = 'external'
    os.floating_ip_pool_always_allocate = true
    os.keypair_name       = 'id_rsa-acceptance_pub'
    os.networks           = ['network1']
    os.security_groups    = ['default']
    os.flavor             = 'vol.small'
    os.volume_boot        = { 'size': '16', 'delete_on_destroy': true, 'image': 'centos_7_x86_64' }
    override.ssh.insert_key = true
    override.ssh.username = 'centos'
    override.vm.synced_folder ".", "/vagrant", type: "rsync",
        rsync__exclude: [".git/", ".bundle", "modules"]
  end

  config.vm.provider :vmpooler do |vmpooler|
    vmpooler.os = "centos-7-x86_64"
    vmpooler.ttl = 4
  end

  config.vm.define 'master' do |node|
    node.vm.hostname = 'master.platform9.puppet.net'
    node.vm.provision :hosts do |hosts|
      hosts.sync_hosts = true
      hosts.preserve_order = true
      hosts.imports = ['global']
      hosts.exports = {
        'global' => [
          ['@vagrant_ssh', ['@vagrant_hostnames']]
        ]
      }
    end
    node.vm.provider :openstack do |os, override|
      os.flavor = 'vol.medium'
    end
    node.bolt.run_as = 'root'
    node.vm.provision :bolt do |bolt|
      bolt.command = :plan
      bolt.name = 'deploy_pe::provision_master'
    end
  end

  config.vm.define 'agent' do |node|
    node.vm.provision :hosts do |hosts|
      hosts.sync_hosts = true
      hosts.preserve_order = true
      hosts.imports = ['global']
    end
    node.vm.provision :bolt do |bolt|
      bolt.command = :plan
      bolt.name = 'deploy_pe::provision_agent'
      bolt.params = {'master' => 'master'}
    end
    node.trigger.before :destroy do |trigger|
      trigger.name = "Removing agent certificates"
      trigger.ruby do |env, machine|
        VagrantBolt.plan('deploy_pe::decom_agent', env, machine, params: {'master' => 'master'})
      end
    end
  end
end
