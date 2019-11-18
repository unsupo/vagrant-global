# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

dir = File.dirname(File.expand_path(__FILE__))

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Install required Vagrant plugins
required_plugins = ["vagrant-hostmanager"]
required_plugins.each do |plugin|
  unless Vagrant.has_plugin?(plugin) || ARGV.include?(plugin)
  system("vagrant plugin install #{plugin}") || exit!
    # Attempt to install plugin. Bail out on failure to prevent an infinite loop.
    system("vagrant plugin install #{plugin}") || exit!

    # Relaunch Vagrant so the plugin is detected. Exit with the same status code.
    exit system('vagrant', *ARGV)
  end
end

def create_machine(data,config,master,ip)
    net_ip = "#{data['net_ip'] || '192.168.1'}"
    global_box = "#{data['box'] || 'bento/ubuntu-16.04' }"
    global_name = "#{global_box}"

    # data is global properties
    # master is machine specific properties

    # if data.has_key?('box')
    #   box = "#{data['box']}"
    # end
    box = "#{global_box}"
    if master.has_key?('box') # if machine has box defined, used that other wise use default
      box = "#{master['box']}"
    end
    if data.has_key?('boxurl')
      boxurl = "#{data['boxurl']}"
    end
    if master.has_key?('boxurl')
      boxurl = "#{master['boxurl']}"
    end
    if data.has_key?('boxversion')
      boxversion = "#{data['boxversion']}"
    end
    if master.has_key?('boxversion')
      boxversion = "#{master['boxversion']}"
    end
    n = "#{global_name}-#{ip}"
    name = "#{master['name'] || n}".gsub('/','-')

    config.vm.define "#{name}" do |master_config|
      master_config.vm.provider "virtualbox" do |vb|
        vb.memory = "#{master['mem'] || 512}"
        vb.cpus = "#{master['cpus'] || 1}"
        vb.name = "#{name}"
        vb.customize ["modifyvm", :id, "--nictype1", "virtio" ]
        vb.customize ["modifyvm", :id, "--nictype2", "virtio" ]
      end # end provider
      # if all masters need these files
      if data.has_key?('synced_folders')
        data['synced_folders'].each do |folder|
          master_config.vm.synced_folder folder['src'], folder['dest'], folder['options']
        end
      end
      # if all masters need these ports
      if data.has_key?('forwarded_ports')
        data['forwarded_ports'].each do |port|
          master_config.vm.network "forwarded_port", guest: port['guest'], host: port['host']
        end
      end
      if data.has_key?('packages')
        data['packages'].each do |pkg|
          master_config.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
              which apt-get && apt-get install -y "#{pkg}" || yum install -y "#{pkg}"
            SHELL
          end # end shell do
        end # end packages do
      end # end if
      if data.has_key?('shell_commands')
        data['shell_commands'].each do |cmd|
          master_config.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
              #{cmd}
            SHELL
          end # end shell do
        end
      end
      if data.has_key?('shell_scripts')
        data['shell_scripts'].each do |script|
          master_config.vm.provision "shell", path: "#{script}"
        end
      end
      master_config.vm.box = "#{box}"
      if master.has_key?('boxversion') || data.has_key?('boxversion')
        master_config.vm.box_version = "#{boxversion}"
      end
      if master.has_key?('boxurl') || data.has_key?('boxurl')
        master_config.vm.box_url = "#{boxurl}"
      end
      master_config.vm.hostname = "#{name}"
      if master.has_key?('ip')
        master_config.vm.network "private_network", ip: "#{net_ip}#{master['ip']}"
      else
        master_config.vm.network "private_network", type: "dhcp"
      end
      if master.has_key?('synced_folders')
        master['synced_folders'].each do |folder|
          master_config.vm.synced_folder folder['src'], folder['dest'], folder['options']
        end
      end
      if master.has_key?('forwarded_ports')
        master['forwarded_ports'].each do |port|
          master_config.vm.network "forwarded_port", guest: port['guest'], host: port['host']
        end
      end
      if master.has_key?('packages')
        master['packages'].each do |pkg|
          master_config.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
              which apt-get && apt-get install -y "#{pkg}" || yum install -y "#{pkg}"
            SHELL
          end # end shell do
        end # end packages do
      end # end if
      if master.has_key?('shell_commands')
        master['shell_commands'].each do |cmd|
          master_config.vm.provision "shell" do |shell|
            shell.inline = <<-SHELL
              #{cmd}
            SHELL
          end # end shell do
        end
      end
      if master.has_key?('shell_scripts')
        master['shell_scripts'].each do |script|
          master_config.vm.provision "shell", path: "#{script}"
        end
      end
    end # end of config.vm.define
end # end create_machine

def configure(data)
  Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = false
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true

    ip = 0
    data['machines'].each do |master|
      if master.has_key?('count')
        v=0
        while v < master['count']
          create_machine(data,config,master,ip)
          v+=1
          ip+=1
        end
        next
      end
      create_machine(data,config,master,ip)
      ip+=1
    end # end of master loop
  end # end of global vagrant config
end

data = {"machines"=>[{"count"=>1}]}
if !File.file?("#{dir}/config.yaml")
  print "No config.yaml file using defaults, to override Please copy config.yaml.example to config.yaml then configure your hosts to continue\n"
  # configure()
  # exit
else
  data = YAML.load_file("#{dir}/config.yaml")
end
configure(data)
