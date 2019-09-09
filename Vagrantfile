# -*- mode: ruby -*-
# vi: set ft=ruby :

# Source: https://github.com/martinanderssondotcom/dev-mini

CONFIGURATION = {
  machines: 'media',
  ansible_group: 'media',
  # https://github.com/martinanderssondotcom/box-ubuntu-budgie-19.04
  box: 'ubuntu/bionic64',
  box_version: "20190903.0.0",
  first_ip: '192.168.1.44',
  cpus: 2,
  #cpus: Etc.nprocessors,
  memory_mb: 4096,
  gui: false,
}



['vagrant-vbguest', 'vagrant-hostmanager'].reject{|p| Vagrant.has_plugin? p}.each do |p|
  if system "vagrant plugin install #{p}"
    exec 'vagrant ' + (ARGV.join ' ')
  else
    abort %Q.Failed to install "#{p}" plugin :'(.
  end
end


require 'psych'


Vagrant.configure('2') do |config|
  config.hostmanager.enabled = true
  config.hostmanager.include_offline = true

  # Prioritize VMware
  # https://www.vagrantup.com/docs/providers/basic_usage.html#default-provider
  config.vm.provider 'vmware_desktop'

  config.vm.provider 'virtualbox' do |vb|
    vb.linked_clone = true
  end
end


def define_machine options
  Vagrant.configure('2') do |config|
    name = options[:name]

    config.vm.define name do |machine|
      machine.vm.box = options[:box]
      machine.vm.box_version = options[:box_version]
      ip = options[:ip]
      #machine.vm.network 'private_network', ip: ip
      machine.vm.network "public_network", ip: ip
      machine.vm.hostname = name

      # VMware
      machine.vm.provider('vmware_desktop') do |v|
        v.vmx['displayname'] = name + " (#{ip})"
        v.vmx['numvcpus'] = options[:cpus]
        v.vmx['memsize'] = options[:memory]
      end

      # VirtualBox
      machine.vm.provider('virtualbox') do |v|
        v.name = name + " (#{ip})"
        v.cpus = options[:cpus]
        v.memory = options[:memory]

        unless options[:gui].equal? nil
          v.gui = options[:gui]
        end
      end
    end
  end
end


machine_names = []

ansible_groups = Hash.new { |hash, key|
  hash[key] = []
}


# Walk through all profiles and feed each machine to define_machine()
(CONFIGURATION.is_a?(Hash) ? [CONFIGURATION] : CONFIGURATION).each do |profile|
  ip_parts = profile[:first_ip].rpartition '.'

  arr = *profile[:machines]
  arr.each.with_index do |name, index|
    define_machine name: name,
      box: profile[:box],
      ip: ip_parts.first + '.' + (ip_parts.last.to_i + index).to_s,
      cpus: profile[:cpus],
      memory: profile[:memory_mb],
      gui: profile[:gui]

    machine_names << name

    grp = profile[:ansible_group]

    unless grp.equal? nil
      ansible_groups[grp] << name
    end
  end
end


# What follows is basically setting up the Ansible controller.
# Some of it explained here: https://stackoverflow.com/a/46045193

controller = machine_names.last

# We must, for each machine we're trynna control, copy over the
# Vagrant-generated private key to the controller and register the key with
# Ansible.
#
# PRIVATE_KEYS is a hash keyed by machine name (excluding the controller
# itself). The value of each entry is another hash with two paths. The old path
# is the path on the host machine and this value is provider-specific; can not
# be parsed in the Vagrantfile at this moment so can not be fully built just
# yet. The new path is the new local path on the Ansible controller.

PRIVATE_KEYS = machine_names.grep_v(controller).reduce({}) do |hash, machine|
  hash[machine] = {
    old: "/vagrant/.vagrant/machines/#{machine}/{{provider}}/private_key",
    new: '/home/vagrant/.ssh/private_key_' + machine }
  hash
end

# The provision_key_copy method will register (rather, overrides; see later
# comment) a shell provisioner which copies over the key from the old path on
# the host machine to the new path on the controller.

def provision_key_copy(config, provider)
  PRIVATE_KEYS.each do |machine, key|
    real_path = key[:old].sub '{{provider}}', provider

    config.vm.provision 'get private key from ' + machine, type: :shell, preserve_order: true, inline: <<~EOM
      cp #{real_path} #{key[:new]}
      chown vagrant:vagrant #{key[:new]}
      chmod 400 #{key[:new]}
    EOM
  end
end

Vagrant.configure('2') do |config|
  config.vm.define controller do |last|

    # We have no access to the provider's name and therefore we must use a
    # provider override to build the old key path. But here comes problem number
    # two: Vagrant's "outside-in" order for provisioners would have meant that
    # the Ansible provisioner executes first before the key has been copied
    # which obviously is not a good thing and we have no way to explicitly set
    # the order of execution.
    #
    # So the only solution here is to preemptively declare the key-copy
    # provisioner before the Ansible provisioner and then let Vagrant's provider
    # override "supply the real implementation" afterwards. Please note that
    # without the "preserve_order" flag in the provisioner override, this hack
    # wouldn't have worked.

    # last.vm.provision 'clear .ansible folder', type: :shell, inline: <<~'EOM'
    #     rm -rf /home/vagrant/.ansible/
    #     rm -rf /home/ubunut/.ansible/
    # EOM

    last.vm.provision "file", source: "./provision", destination: "provision"
    last.vm.provision "file", source: ".env.example", destination: ".env"
    last.vm.provision "file", source: "docker-compose.yml", destination: "docker-compose.yml"


    PRIVATE_KEYS.each_key { |machine|
      last.vm.provision 'get private key from ' + machine, type: :shell,
        inline: 'echo Your stupid provider is not supported. Good luck bro.; exit 1;'
    }

    ['virtualbox', 'vmware_desktop'].each { |p|
      last.vm.provider p do |x, override|
        provision_key_copy override, p
      end
    }

    last.vm.provision 'ansible', type: :ansible_local do |ansible|
      ansible.playbook = '/home/vagrant/provision/playbook.yaml'
      ansible.limit = 'all'

      ansible.host_vars = {}

      PRIVATE_KEYS.each do |machine, key|
        ansible.host_vars[machine] = {
          'ansible_ssh_private_key_file' => key[:new],
          'ansible_ssh_extra_args' => '-o StrictHostKeyChecking=no'
        }
      end

      ansible.groups = ansible_groups

      # I use pip args only to stop Vagrant from tossing in the upgrade flag. Not
      # sure wtf that flag does. See:
      # https://www.vagrantup.com/docs/provisioning/ansible_local.html#options
      ansible.install_mode = :pip_args_only
      ansible.pip_args = 'ansible==2.8.0'

      roles_file = '/home/vagrant/provision/requirements.yaml'

      if File.exist?(roles_file) && !Psych.load_file(roles_file, nil).equal?(nil)
        ansible.galaxy_role_file = roles_file
        ansible.galaxy_roles_path = '/home/vagrant/.ansible/roles/'
        ansible.galaxy_command = 'ansible-galaxy install --role-file=%{role_file} --roles-path=%{roles_path}'
      end
    end
  end
end

#$script = <<-SCRIPT
#apt-get update
#apt-get install docker docker-compose -y
#mkdir -p /media/my_user/storage/homemedia
#chown -R vagrant:vagrant /media/my_user/storage/homemedia
#systemctl enable docker && systemctl start docker
#docker-compose up -d
#usermod -aG docker vagrant
#echo 192.168.1.44    organizr >> /etc/hosts
#SCRIPT


#
# Vagrant.configure("2") do |config|
#   config.vm.box = "ubuntu/bionic64"
#
#   config.vm.provider "virtualbox" do |v|
#     v.memory = 2048
#     v.cpus = 2
#   end
#
#   config.vm.provision "file", source: ".env.example", destination: ".env"
#   config.vm.provision "file", source: "docker-compose.yml", destination: "docker-compose.yml"
#   #config.vm.provision "file", source: "./provision", destination: "/home/vagrant/provision"
#   #config.vm.provision "shell", inline: $script
#
#   config.vm.provision 'preemptively give others write access to /etc/ansible/roles', type: :shell, inline: <<~'EOM'
#     sudo mkdir /usr/share/ansible/modules/roles -p
#     sudo chmod o+w /usr/share/ansible/modules/roles
#     sudo chown -R vagrant:vagrant /usr/share/ansible
#   EOM
#
#   config.vm.provision 'ansible', run: 'always', type: :ansible_local do |ansible|
#     ansible.galaxy_role_file = 'provision/requirements.yaml'
#     ansible.galaxy_roles_path = '/usr/share/ansible/modules/roles'
#     ansible.galaxy_command = 'ansible-galaxy install --role-file=%{role_file} --roles-path=%{roles_path}'
#     ansible.playbook = 'provision/playbook.yaml'
#     ansible.verbose = true
#   end
#
#   config.vm.network "public_network", ip: "192.168.1.44"
# end
