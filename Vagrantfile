# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

if not File.file?("#{File.dirname(__FILE__)}/.vagrant/id_rsa")
  system("
    ssh-keygen -f './.vagrant/id_rsa' -t rsa -N '' -C 'vagrant@vagrant'
  ")
end

if File.file?("#{File.dirname(__FILE__)}/vagrantconf.yml")
  configuration = YAML::load(File.read("#{File.dirname(__FILE__)}/vagrantconf.yml"))
else
  configuration = YAML::load(File.read("#{File.dirname(__FILE__)}/vagrantconf.default.yml"))
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|

  #
  # CONFIGURE SECOND SERVER IF envtype == 2app
  # envtype=2app vagrant up
  #
  if configuration.key?("app2")

    config.vm.define "app2" do |app2|

      # app2.vm.box = "centos/7"
      app2.vm.box = "bento/centos-7.3"
      # app2.vm.box = "geerlingguy/centos7"
      app2.vm.hostname = 'app2'
      # app2.vm.box_url = "ubuntu/precise64"

      app2.vm.network :private_network, ip: "192.168.56.57"

      app2.vm.provider :virtualbox do |v|
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        v.customize ['modifyvm', :id, '--cableconnected1', 'on']
        v.customize ["modifyvm", :id, "--memory", configuration["app2"]["memory"] ]
        v.customize ["modifyvm", :id, "--cpus", configuration["app2"]["cpus"] ]
        v.customize ["modifyvm", :id, "--name", "app2"]
      end

      # Non-controlling server should not have meza
      app2.vm.synced_folder ".", "/vagrant", disabled: true
   	  # app2.vm.synced_folder ".", "/opt/meza", type: "rsync",
      #   rsync__args: ["--verbose", "--archive", "--delete", "-z"]

      # Transfer setup-minion-user.sh script to app2
      app2.vm.provision "file", source: "./src/scripts/ssh-users/setup-minion-user.sh", destination: "/tmp/minion.sh"
      app2.vm.provision "file", source: "./.vagrant/id_rsa.pub", destination: "/tmp/meza-ansible.id_rsa.pub"

      #
      # Setup SSH user and unsafe testing config
      #
      app2.vm.provision "minion-ssh", type: "shell", preserve_order: true, binary: true, inline: <<-SHELL
        if [ ! -f /opt/conf-meza/public/vars.yml ]; then

          bash /tmp/minion.sh

          # Turn off host key checking for user meza-ansible, to avoid prompts
          echo "setup .ssh/config"
          bash -c 'echo -e "Host *\n   StrictHostKeyChecking no\n   UserKnownHostsFile=/dev/null" > /opt/conf-meza/users/meza-ansible/.ssh/config'
          sudo chown meza-ansible:meza-ansible /opt/conf-meza/users/meza-ansible/.ssh/config
          sudo chmod 600 /opt/conf-meza/users/meza-ansible/.ssh/config

          # Allow password auth
          echo "setup sshd_config password auth"
          sed -r -i 's/PasswordAuthentication no/PasswordAuthentication yes/g;' /etc/ssh/sshd_config
          systemctl restart sshd
        fi
      SHELL
    end

  end

  config.vm.define "app1", primary: true do |app1|

    # app1.vm.box = "centos/7"
    app1.vm.box = "bento/centos-7.3"
    # app1.vm.box = "geerlingguy/centos7"
    app1.vm.hostname = 'app1'
    # app1.vm.box_url = "ubuntu/precise64"

    app1.vm.network :private_network, ip: "192.168.56.56"

    app1.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ['modifyvm', :id, '--cableconnected1', 'on']
      v.customize ["modifyvm", :id, "--memory", configuration["app1"]["memory"] ]
      v.customize ["modifyvm", :id, "--cpus", configuration["app1"]["cpus"] ]
      v.customize ["modifyvm", :id, "--name", "app1"]
    end

    # Disable default synced folder at /vagrant, instead put at /opt/meza
    app1.vm.synced_folder ".", "/vagrant", disabled: true
    app1.vm.synced_folder ".", "/opt/meza", type: "virtualbox"
    # app1.vm.synced_folder ".", "/opt/meza", type: "smb"
    # app1.vm.synced_folder ".", "/opt/meza", type: "rsync",
    #   rsync__args: ["--verbose", "--archive", "--delete", "-z"]


    # Transfer keys to app1
    app1.vm.provision "file", source: "./.vagrant/id_rsa", destination: "/tmp/meza-ansible.id_rsa"
    app1.vm.provision "file", source: "./.vagrant/id_rsa.pub", destination: "/tmp/meza-ansible.id_rsa.pub"


    #
    # Bootstrap meza on the controlling VM
    #
    app1.vm.provision "getmeza", type: "shell", preserve_order: true, inline: <<-SHELL
      bash /opt/meza/src/scripts/getmeza.sh
      rm -rf /opt/conf-meza/users/meza-ansible/.ssh/id_rsa
      rm -rf /opt/conf-meza/users/meza-ansible/.ssh/id_rsa.pub
      mv /tmp/meza-ansible.id_rsa /opt/conf-meza/users/meza-ansible/.ssh/id_rsa
      mv /tmp/meza-ansible.id_rsa.pub /opt/conf-meza/users/meza-ansible/.ssh/id_rsa.pub

      chmod 600 /opt/conf-meza/users/meza-ansible/.ssh/id_rsa
      chown meza-ansible:meza-ansible /opt/conf-meza/users/meza-ansible/.ssh/id_rsa
      chmod 644 /opt/conf-meza/users/meza-ansible/.ssh/id_rsa.pub
      chown meza-ansible:meza-ansible /opt/conf-meza/users/meza-ansible/.ssh/id_rsa.pub

      cat /opt/conf-meza/users/meza-ansible/.ssh/id_rsa.pub >> /opt/conf-meza/users/meza-ansible/.ssh/authorized_keys
    SHELL

    #
    # Setup meza environment, either monolithic or 2app
    #
    if not configuration.key?("app2")

      app1.vm.provision "setupenv", type: "shell", preserve_order: true, inline: <<-SHELL
        meza setup env vagrant --fqdn=192.168.56.56 --db_pass=1234 --private_net_zone=public
      SHELL

    else

      # FIXME: should have a less fragile method to check if env exists
      app1.vm.provision "setupenv", type: "shell", preserve_order: true, inline: <<-SHELL
        if [ ! -d /opt/conf-meza/secret/vagrant ]; then
          default_servers=192.168.56.56 app_servers=192.168.56.56,192.168.56.57 meza setup env vagrant --fqdn=192.168.56.56 --db_pass=1234 --private_net_zone=public
        fi
      SHELL
    end

    #
    # Setup meza public config
    #
    app1.vm.provision "publicconfig", type: "shell", preserve_order: true, inline: <<-SHELL
      # Create public config dir if not exists
      [ -d /opt/conf-meza/public ] || mkdir /opt/conf-meza/public

      # If public config YAML file not present, create with defaults
      if [ ! -f /opt/conf-meza/public/vars.yml ]; then
cat >/opt/conf-meza/public/vars.yml <<EOL
---
blender_landing_page_title: Meza Wikis
m_setup_php_profiling: true
m_force_debug: true
EOL
      fi

      cat /opt/conf-meza/public/vars.yml
    SHELL

    #
    # If multi-app: transfer key to second app server
    #
    if configuration.key?("app2")



      app1.vm.provision "keytransfer", type: "shell", preserve_order: true, inline: <<-SHELL

        # Turn off host key checking for user meza-ansible, to avoid prompts
        bash -c 'echo -e "Host *\n   StrictHostKeyChecking no\n   UserKnownHostsFile=/dev/null" > /opt/conf-meza/users/meza-ansible/.ssh/config'
        sudo chown meza-ansible:meza-ansible /opt/conf-meza/users/meza-ansible/.ssh/config
        sudo chmod 600 /opt/conf-meza/users/meza-ansible/.ssh/config

        # Allow SSH login
        # WARNING: This is INSECURE and for test environment only
        sed -r -i 's/UsePAM yes/UsePAM no/g;' /etc/ssh/sshd_config
        systemctl restart sshd

        echo "switch user"
        sudo su meza-ansible

        # Copy id_rsa.pub to each minion
        # sshpass -p 1234 ssh meza-ansible@192.168.56.57 "echo \"$pubkey\" >> /opt/conf-meza/users/meza-ansible/.ssh/authorized_keys"


        # Remove password-based authentication for $ansible_user
        #echo "delete password"
        #ssh 192.168.56.57 "sudo passwd --delete meza-ansible"

        # Allow SSH login
        # WARNING: This is INSECURE and for test environment only
        # echo "setup sshd_config"
        # ssh 192.168.56.57 "sudo sed -r -i 's/UsePAM yes/UsePAM no/g;' /etc/ssh/sshd_config && sudo systemctl restart sshd"
      SHELL

    else

      #
      # Finally: Deploy if not multi-server
      #
      app1.vm.provision "deploy", type: "shell", preserve_order: true, inline: <<-SHELL
        meza deploy vagrant
      SHELL
    end

  end

end
