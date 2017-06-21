# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANT_ARGS = ARGV

# config_file = 'vm-config.yml-example'

# Shoehorn the config file into our test Vagrantfile
require_relative File.join(File.expand_path(Dir.pwd), 'templates/class_config.rb')

dreambox_config_file = (defined?(config_file)) ? config_file : 'vm-config.yml'
dns_hosts_file = File.join(File.expand_path(Dir.pwd), 'templates/dns_hosts.txt')

Dreambox = Config.new(dreambox_config_file, dns_hosts_file)

Vagrant.configure(2) do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.box_url = "https://atlas.hashicorp.com/hashicorp/boxes/precise64"

  config.vm.network :forwarded_port, guest: 80, host: 8080, auto_correct: true

  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end

  # Set these so the provisioning scripts can be run via ssh
  config.vm.synced_folder "files", "/tmp/files", create: false, :mount_options => ["dmode=775", "fmode=664"]
  config.vm.synced_folder "packages", "/tmp/packages", create: false, :mount_options => ["dmode=775", "fmode=664"]

  # REVIEW See synced_folder line
  # $user_vars["DREAMBOX_UID"] = 31415

  # REVIEW We're going to listen for a user _after_ `ssh` instead
  # REVIEW We need to check for multimachine so shit doesn't break
  # if 'ssh' == VAGRANT_ARGS.first && VAGRANT_ARGS.length == 3
  #   config.ssh.username = VAGRANT_ARGS[1] if config.vm.defined_vm_keys.length == 1
  #   puts 'logging in with user'
  # end

  # Development machine
  # Ubuntu 12.04
  config.vm.define 'dev', autostart: false do |dev|
    dev.vm.hostname = "dreambox.dev"
    dev.vm.network :private_network, ip: "192.168.12.34"
  end

  # Testing machine
  # Fully provisioned and ready to test
  config.vm.define 'test', primary: true do |test|
    test.vm.hostname = "dreambox.test"
    test.vm.network :private_network, ip: "192.168.56.78"


    # Start bash as a non-login shell
    test.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

    # Installed utinities and libraries
    test.vm.provision "base",
      type: "shell",
      path: "scripts/base.sh",
      # Environment variable for simulating Packer file provisioner
      :env => {"DREAMBOX_ENV" => "develop"}

    # Post-install MySQL setup
    test.vm.provision "package-setup",
      type: "shell",
      path: "scripts/package-setup.sh"

    # Install PHP
    test.vm.provision "shell",
      inline: "/bin/bash /usr/local/bin/php_install",
      :env => Dreambox.config

    if Dreambox.config['ssl_enabled'] then
      test.vm.provision "shell",
        inline: "/bin/bash /usr/local/bin/ssl_setup",
        :env => Dreambox.config
    end

    # TODO Change this to .each_value
    Dreambox.config['sites'].each do |site, conf|
      if (! conf['is_subdomain']) then
        # Sets up the sync folder
        test.vm.synced_folder conf['local_root'], conf['root_path'],
          # REVIEW Generate the UID, pass it to `user_setup` with conf
          owner: conf['uid'],
          group: "www-data",
          mount_options: ["dmode=775,fmode=664"]
      end
      # Runs user_setup
      test.vm.provision "shell",
        inline: "/bin/bash /usr/local/bin/user_setup",
        :env => conf
    end
  end
end
