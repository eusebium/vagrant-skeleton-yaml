require 'rbconfig'
require 'yaml'

# set default LC_ALL for all BOXES
ENV["LC_ALL"] = "en_US.UTF-8"

# Set your default base box here
DEFAULT_BASE_BOX = 'bento/centos-7.6'

# When set to `true`, Ansible will be forced to be run locally on the VM
# instead of from the host machine (provided Ansible is installed).
FORCE_LOCAL_RUN = false

VAGRANTFILE_API_VERSION = '2'
PROJECT_NAME = File.basename(Dir.getwd).upcase

# set custom vagrant-hosts file
vagrant_hosts = ENV['VAGRANT_HOSTS'] ? ENV['VAGRANT_HOSTS'] : 'vagrant-hosts.yml'
hosts = YAML.load_file(File.join(Dir.pwd, vagrant_hosts))

def run_locally?
  windows_host? || FORCE_LOCAL_RUN
end

def windows_host?
  Vagrant::Util::Platform.windows?
end


# Set options for the network interface configuration. All values are
# optional, and can include:
# - ip (default = DHCP)
# - netmask (default value = 255.255.255.0
# - mac
# - auto_config (if false, Vagrant will not configure this network interface
# - intnet (if true, an internal network adapter will be created instead of a
#   host-only adapter)
def network_options(host)
  options = {}

  if host.key?('ip')
    options[:ip] = host['ip']
    options[:netmask] = host['netmask'] ||= '255.255.255.0'
  else
    options[:type] = 'dhcp'
  end

  options[:mac] = host['mac'].gsub(/[-:]/, '') if host.key?('mac')
  options[:auto_config] = host['auto_config'] if host.key?('auto_config')
  options[:virtualbox__intnet] = true if host.key?('intnet') && host['intnet']
  options
end


def synced_folders(vm, host)
  return unless host.key?('synced_folders')
  folders = host['synced_folders']

  folders.each do |folder|
    vm.synced_folder folder['src'], folder['dest'], folder['options']
  end
end


def extra_disks(vm, host)
  return unless host.key?('extra_disks')
  extra_disks = host['extra_disks']
  vb_property_default_machine_folder = `VBoxManage list systemproperties | grep "Default machine folder"`
  vb_machine_folder = vb_property_default_machine_folder.split(':')[1].strip()
  i = 2

  extra_disks.each do |extra_disk|
    extra_disk_path = File.join(vb_machine_folder, vm.name, 'disk' + i.to_s + '.vdi')
    unless File.exist?(extra_disk_path)
      vm.customize ['createhd',
        '--filename', extra_disk_path,
        '--size', extra_disk.to_i]
    end
    vm.customize ['storageattach', :id,
      '--storagectl', 'SATA Controller',
      '--port', i,
      '--type', 'hdd',
      '--medium', extra_disk_path,
      '--nonrotational', 'on']
    i += 1
  end
end


def shared_disks(vm, host)
  return unless host.key?('shared_disks')
  shared_disks = host['shared_disks']
  shared_disks_path = "./"
  i = 10
  shared_disks.each do |shared_disk|
    shared_disk_path = shared_disks_path + shared_disk['name'] + ".vdi"
    unless File.exist?(shared_disk_path)
      vm.customize ['createhd',
        '--filename',shared_disk_path,
        '--size', shared_disk['size'].to_i,
        '--format', 'VDI',
        '--variant', 'Fixed']
    end
    vm.customize ['storageattach', :id,
      '--storagectl', 'SATA Controller',
      '--port', i,
      '--type', 'hdd',
      '--medium', shared_disk_path,
      '--nonrotational', 'on',
      '--mtype', 'shareable']
    vm.customize ['modifymedium',
      'disk', shared_disk_path,
      '--type', 'shareable']
    i += 1
  end
end


def attach_iso(vm, host)
  return unless host.key?('attach_iso')
  attach_iso = host['attach_iso']
  i = 10
  attach_iso.each do |isofile|
    vm.customize ["storageattach", :id,
      "--storagectl", "SATA Controller",
      "--port", i,
      "--type", "dvddrive",
      "--medium", isofile]
    i += 1
  end
end


# Set options for shell provisioners to be run always. If you choose to include
# it you have to add a cmd variable with the command as data.
#
# Use case: start symfony dev-server
#
# example:
# shell_always:
#   - cmd: php /srv/google-dev/bin/console server:start 192.168.52.25:8080 --force
def shell_provisioners_always(vm, host)
  if host.has_key?('shell_always')
    scripts = host['shell_always']

    scripts.each do |script|
      vm.provision "shell", inline: script['cmd'], run: "always"
    end
  end
end

# }}}


# Set options for shell provisioners to be run always. If you choose to include
# it you have to add a cmd variable with the command as data.
#
# example:
# trigger_before:
#   - when: :up
#     cmd: ansible-playbook mount_iso.yml -e publish_state_play=False -e publish_path_iso_files=file.iso
def trigger_before(node, host)
  if host.has_key?('trigger_before')
    triggers = host['trigger_before']

    triggers.each do |host_trigger|
      node.trigger.before host_trigger['when'] do |trigger|
        trigger.run = {inline: host_trigger['cmd']}
      end
    end
  end
end

# }}}


# Set options for shell provisioners to be run always. If you choose to include
# it you have to add a cmd variable with the command as data.
#
# example:
# trigger_before:
#   - when: :up
#     cmd: ansible-playbook mount_iso.yml -e publish_state_play=False -e publish_path_iso_files=file.iso
def trigger_after(node, host)
  if host.has_key?('trigger_after')
    triggers = host['trigger_after']

    triggers.each do |host_trigger|
      node.trigger.after host_trigger['when'] do |trigger|
        trigger.run = {inline: host_trigger['cmd']}
      end
    end
  end
end


# Adds forwarded ports to your Vagrant machine
#
# example:
#  forwarded_ports:
#    - guest: 88
#      host: 8080
def forwarded_ports(vm, host)
  if host.has_key?('forwarded_ports')
    ports = host['forwarded_ports']

    ports.each do |port|
      vm.network "forwarded_port", guest: port['guest'], host: port['host']
    end
  end
end


def provision_ansible(node, host)
  ansible_mode = run_locally? ? 'ansible_local' : 'ansible'
  return unless host.key?('playbooks')
  playbooks = host['playbooks']
  playbooks.each do |playbook|
    node.vm.provision ansible_mode do |ansible|
      ansible.compatibility_mode = '2.0'
      if playbook.kind_of?(String)
        ansible.playbook = playbook
      else
        ansible.playbook = playbook['playbook']
        ansible.extra_vars = playbook['extra_vars'] if playbook.key? 'extra_vars'
        if playbook.has_key?('groups')
          groups = playbook['groups']
          groups.each do |group|
            ansible.groups = {
              "#{group}": host['name'],
              "redhat5:vars": {
                "ansible_sudo_flags": "-H",
                "ansible_python_interpreter": "/usr/bin/python2.6",
              },
            }
          end
        end
      end
    end
  end
end


# Add to all vms a storage controller
class VagrantPlugins::ProviderVirtualBox::Action::SetName
  alias_method :original_call, :call
  def call(env)
    machine = env[:machine]
    driver = machine.provider.driver
    uuid = driver.instance_eval { @uuid }
    ui = env[:ui]

    controller_name = "SATA Controller"

    vm_info = driver.execute("showvminfo", uuid)
    has_this_controller = vm_info.match("Storage Controller Name.*#{controller_name}")

    if has_this_controller
      ui.info "Already has the #{controller_name}"
    else
      ui.info "Creating #{controller_name}"
      driver.execute('storagectl', uuid,
        '--name', "#{controller_name}",
        '--add', 'sata',
        '--controller', 'IntelAhci')
    end
    original_call(env)
  end
end


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box_check_update = false
  config.ssh.insert_key = false

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  if Vagrant.has_plugin?("vagrant-proxyconf")
    config.proxy.enabled = { docker: false }
  end

  if Vagrant.has_plugin?("vagrant-timezone")
    config.timezone.value = :host
  end

  hosts.each do |host|
    config.vm.define host['name'] do |node|
      node.vm.box = host['box'] ||= DEFAULT_BASE_BOX
      node.vm.box_url = host['box_url'] if host.key? 'box_url'
      node.vm.box_version = host['box_version'] if host.key? 'box_version'
      if host.key?('domain')
        node.vm.hostname = host['name'] + '.' + host['domain']
      else
        node.vm.hostname = host['name']
      end
      node.vm.network :private_network, network_options(host)
      forwarded_ports(node.vm, host)
      trigger_before(node, host)
      trigger_after(node, host)
      synced_folders(node.vm, host)
      shell_provisioners_always(node.vm, host)
      provision_ansible(node, host)

      node.vm.provider :virtualbox do |vb|
        vb.memory = host['memory'] if host.key? 'memory'
        vb.cpus = host['cpus'] if host.key? 'cpus'
        vb.name = PROJECT_NAME.downcase + '_' + host['name']
        attach_iso(vb, host)
        extra_disks(vb, host)
        shared_disks(vb, host)
      end

    end
  end
end

# -*- mode: ruby -*-
# vi: ft=ruby :
