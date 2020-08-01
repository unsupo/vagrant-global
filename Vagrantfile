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
def provision(dict, master_config)
  if dict.has_key?('shell_commands')
    dict['shell_commands'].each do |cmd|
      master_config.vm.provision "shell" do |shell|
        shell.inline = <<-SHELL
          #{cmd}
        SHELL
      end # end shell do
    end
  end
end
def provision_script(dict,master_config)
  if dict.has_key?('shell_scripts')
    dict['shell_scripts'].each do |script|
      master_config.vm.provision "shell", path: "#{script}"
    end
  end
end
def package(dict, master_config)
  if dict.has_key?('packages')
    dict['packages'].each do |pkg|
      master_config.vm.provision "shell" do |shell|
        shell.inline = <<-SHELL
          which apt-get && apt-get install -y "#{pkg}" || yum install -y "#{pkg}"
        SHELL
      end # end shell do
    end # end packages do
  end # end if
end
def forward_ports(dict,master_config) # TODO: support 8080:8081 syntax
  if dict.has_key?('forwarded_ports')
    if dict['forwarded_ports'].instance_of? Array
      dict['forwarded_ports'].each do |port|
        if port.instance_of?(Hash) && port.has_key?('guest')
          master_config.vm.network "forwarded_port", guest: port['guest'], host: port['host']
        else
          master_config.vm.network "forwarded_port", guest: port, host: port
        end
      end
    else
      master_config.vm.network "forwarded_port", guest: dict['forwarded_ports'], host: dict['forwarded_ports']
    end
  end
end
def sync_folders(dict, master_config)
  if dict.has_key?('synced_folders')
    dict['synced_folders'].each do |folder|
      master_config.vm.synced_folder folder['src'], folder['dest'], folder['options']
    end
  end
end

def create_machine(data,config,master,ip)
    dirname = File.basename(Dir.getwd)
    net_ip = "#{data['net_ip'] || '192.168.1'}"
    global_box = "#{data['box'] || 'bento/ubuntu-16.04' }"
    # might use substring of shasum of absolute path to guarantee uniqueness of box name for this directory without a huge name
    global_name = "#{dirname}-#{global_box}"

    # data is global properties
    # master is machine specific properties

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
      forward_ports(data,master_config)
      forward_ports(master,master_config)
      package(data,master_config)
      package(master,master_config)
      provision(data,master_config)
      provision(master,master_config)
      provision_script(data,master_config)
      provision_script(master,master_config)
      sync_folders(data, master_config)
      sync_folders(master, master_config)
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
        # TODO: Loop over the children of the count and add configs based off it
        # - count:
        #   amount: 4
        #   - n: 2 # this is 0 indexed
        #     forwarded_ports: 8080
        count = master['count']
        # vms = Array.new(count)
        # if master['count'].instance_of? Hash
        #   # if it's an instance of Hash then that means it applies to all in this count
        #   master['count'].each do |key, value|
        #     # add all these key values to below create machine command
        #   end
        # elsif master['count'].instance_of? Array
        #   # if it's an instance of Array then apply it indivdually unless n is all
        #   master['count'].each do |machine|
        #     # use create_machine command and add any key values associated with this machine
        #     # to the create_machine command then set it's n value to true in the vms array
        #     # so the below create_machine doesn't create it again
        #   end
        # end
        v=0
        while v < count
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
  File.open('config.yaml', 'w') { |file| file.write(data.to_yaml) }
  # configure()
  # exit
else
  data = YAML.load_file("#{dir}/config.yaml")
end
configure(data)
