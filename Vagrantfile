# -*- mode: ruby -*-
# vi: set ft=ruby ts=2 sw=2 :

require 'json'
require 'open3'
require 'set'
require 'yaml'

VAGRANTFILE_API_VERSION = "2"

# Ensure libvirt is the default provider in case the vagrant box config
# doesn't specify it.
ENV['VAGRANT_DEFAULT_PROVIDER'] = "libvirt"

# Uncomment the following line if debugging box issues to ensure that
# boxes are started up serially
#ENV['VAGRANT_NO_PARALLEL'] = 'yes'

# Useful paths
@dev_env_root = File.dirname(File.expand_path(__FILE__))
@dev_env_cfg_file = File.join(@dev_env_root, "settings.yml")
@dev_env_ansible = File.join(@dev_env_root, "ansible")
@dev_env_inventory = File.join(@dev_env_ansible, "dev_env_inventory.yml")

# Check for required config files
_required_config_files_list = [@dev_env_cfg_file]
exit(1) unless _required_config_files_list.all? do |cfg_file|
  File.exists?(cfg_file) || (
    STDERR.puts "Missing config file: #{cfg_file}"
    STDERR.puts "Please follow the instructions in the README.md to prepare the"
    STDERR.puts "development environment for use."
    false
  )
end

# Check for required vagrant plugin
_required_plugins_list = %w{vagrant-libvirt}
exit(1) unless _required_plugins_list.all? do |plugin|
  Vagrant.has_plugin?(plugin) || (
    STDERR.puts "Required plugin '#{plugin}' is missing; please install using:"
    STDERR.puts "  % vagrant plugin install #{plugin}"
    false
  )
end

#
# Config Settings
#
@dev_env_cfg = YAML.load_file(@dev_env_cfg_file)
@dev_env_tweaks = @dev_env_cfg.fetch('tweaks', {})

#
# Ansible inventory integration
#
# The following code leverages the ansible-inventory command to process
# the specified inventory, which automatically pulls in the associated
# host_vars and group_vars data, and presents the result in JSON format,
# with the special '_meta' section containing the 'hostvars' settings
# for each node in the inventory.
@_ansible_inventory = nil
def ansible_inventory()
  if !@_ansible_inventory.nil?
    return @_ansible_inventory
  end

  out, err, sts = Open3.capture3("ansible-inventory", "-i", @dev_env_inventory, "all", "--list")
  if sts != 0
    STDERR.puts "Failed to load the ansible inventory:"
    STDERR.puts "#{out}"
    STDERR.puts "#{err}"
    exit(1)
  end

  @_ansible_inventory = JSON.parse(out)
end
#STDERR.puts "ansible_inventory = #{ansible_inventory.inspect}"

def ansible_hostvars()
  return ansible_inventory['_meta']['hostvars']
end

def ip_addr(name)
  return ansible_hostvars[name]['ip']
end

# Determine the flattened group => host list associations
# expose as @ansible_groups
@_ansible_groups = nil
def ansible_groups()
  if !@_ansible_groups.nil?
    return @_ansible_groups
  end

  groups = {}

  def process_group(group_name, groups)
    group_entry = ansible_inventory.fetch(group_name, {})
    if group_entry.key?("hosts")
      groups[group_name] = groups.fetch(group_name, []).to_set.merge(group_entry["hosts"].to_set).to_a
    end
    group_entry.fetch("children", {}).each do |child_group|
      process_group(child_group, groups)
      groups[group_name] = groups.fetch(group_name, []).to_set.merge(groups.fetch(child_group, [])).to_a
    end
  end

  process_group("all", groups)

  # Add the python group
  #ansible_hostvars.each do |hostname, host_vars|
  #  python_group = host_vars["box"]["python"]
  #  groups[python_group] = groups.fetch(python_group, []).append(hostname)
  #end

  @_ansible_groups = groups
end
#STDERR.puts "ansible_groups = #{ansible_groups.inspect}"

# Determine which groups a host is a member of
@_ansible_host_groups = nil
def ansible_host_groups()
  if !@_ansible_host_groups.nil?
    return @_ansible_host_groups
  end

  host_groups = {}

  ansible_groups.each do |group_name, hosts|
    hosts.each do |host|
      host_groups[host] = host_groups.fetch(host, []).append(group_name)
    end
  end

  @_ansible_host_groups = host_groups
end
#STDERR.puts "ansible_host_groups = #{ansible_host_groups.inspect}"


#
# Libvirt node domain setup
#
def configure_libvirt_domain(domain, cfg)
    # Extract relevant config settings, with sensible defaults if not specified
    disk_cfg = cfg.fetch('disks', {})
    boot_cfg = disk_cfg.fetch('boot', {'cache' => 'unsafe'})
    cpus_cfg = cfg.fetch('cpus', 2)
    mtu_cfg = @dev_env_tweaks.fetch('management_network_mtu', nil)

    # Vagrant integration
    domain.default_prefix = ""

    # Fix the Vagrant management network mtu if needed
    if mtu_cfg and domain.respond_to?('management_network_mtu')
      domain.management_network_mtu = mtu_cfg
    end

    # CPU & Memory settings
    domain.memory = cfg.fetch('memory', 4096)
    if cpus_cfg.is_a?(Hash)
      domain.cputopology :sockets => cpus_cfg['sockets'], :cores => cpus_cfg['cores'], :threads => cpus_cfg['threads']
    else
      domain.cpus = cpus_cfg
    end

    # primary disk
    if boot_cfg.key?('size')
      domain.machine_virtual_size = boot_cfg['size']
    end
    domain.disk_bus = cfg.fetch("disk_bus", "virtio")
    if domain.respond_to?('disk_driver')
      domain.disk_driver :cache => boot_cfg['cache']
    else
      domain.volume_cache = boot_cfg['cache']
    end

    # management network
    #domain.management_network_name = "#{@dev_env_cfg['libvirt']['mgmt']['name']}-libvirt"
    #domain.management_network_address = @dev_env_cfg['libvirt']['mgmt']['net']

    # KVM
    domain.nested = true
    domain.cpu_mode = 'host-passthrough'

    # Graphics
    domain.graphics_type = "vnc"
    #domain.graphics_ip = "0.0.0.0"  # default is 127.0.0.1

    # Add extra disks
    if disk_cfg.key?("extras")
      disk_cfg["extras"].each do |device, device_def|
        domain.storage :file, :size => device_def["size"], :device => device, :cache => device_def["cache"]
      end
    end
end


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  ansible_hostvars.each do |hostname, host_vars|

    #STDERR.puts "DBG: Host: #{hostname.inspect}, Vars: #{host_vars.inspect}"

    host_vars.fetch("forwarded_ports", []).each do |fport|
      if fport.key?("host_ip")
        config.vm.network "forwarded_port",
          guest: fport["guest"],
          host: fport["host"],
          host_ip: fport["host_ip"]
      else
        config.vm.network "forwarded_port",
          guest: fport["guest"],
          host: fport["host"]
      end
    end

    config.vm.define hostname,
      autostart: host_vars.fetch("autostart", true),
      primary: host_vars.fetch("primary", true) do |node|

      node.vm.hostname = hostname

      node.vm.box = host_vars["box"]
      node.vm.box_check_update = host_vars.fetch("box_check_update", true)

      node.vm.network "private_network", type: "dhcp" , ip: host_vars["ansible_host"]

      # disable default synced folder
      node.vm.synced_folder ".", "/vagrant", disabled: true

      host_vars.fetch("synced_folders", []).each do |sfolder|
        folder_type = sfolder.fetch("type")
        if folder_type and folder_type == "nfs"
          node.vm.synced_folder sfolder["src"], sfolder["dest"],
            disabled: !sfolder.fetch("enabled", false),
            type: "nfs", nfs_udp: false
        else
          node.vm.synced_folder sfolder["src"], sfolder["dest"],
            disabled: !sfolder.fetch("enabled", false)
        end
      end

      # configure libvirt domain specified settings
      node.vm.provider :libvirt do |domain|
        configure_libvirt_domain(domain, host_vars)
      end

      #
      # Provisioning
      #

      # files
      host_vars.fetch("provision_files", []).each do |prov_file|
        node.vm.provision "file", source: prov_file["src"],
          destination: prov_file["dest"]
      end

      # tweaks - fix netplan dns settings
      if @dev_env_tweaks.fetch("fix_netplan_dns", []).include? host_vars["box"]
        node.vm.provision "fix_netplan_dns", type: :shell do |shell|
          netplan_cfg = "/etc/netplan/01-netcfg.yaml"
          shell.inline = <<-SHELL
            grep -e nameservers: -e addresses: #{netplan_cfg} >/dev/null || exit 0
            sudo sed -i -e '/nameservers:/d' -e '/addresses:/d' #{netplan_cfg}
            sudo netplan generate && sudo netplan apply
            sudo sed -i 's/^[[:alpha:]]/#&/' /etc/systemd/resolved.conf
            sudo systemctl restart systemd-resolved.service
          SHELL
        end
      end

      # tweaks - fix interfaces dns settings
      if @dev_env_tweaks.fetch("fix_interfaces_dns", []).include? host_vars["box"]
        node.vm.provision "fix_interfaces_dns", type: :shell do |shell|
          interfaces_cfg = "/etc/network/interfaces"
          shell.inline = <<-SHELL
            grep -e '^dns-nameserver ' #{interfaces_cfg} >/dev/null || exit 0
            sudo sed -i -e '/^dns-nameserver /d' #{interfaces_cfg}
          SHELL
        end
      end

      # main Ansible provisioning
      node.vm.provision :ansible do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.playbook = host_vars.fetch("ansible_playbook",
                                           "#{@dev_env_ansible}/provision.yml")
        ansible.groups = ansible_groups
        #ansible.host_vars = ansible_hostvars
        #ansible.limit = "all"  # need to guard so only runs once all hosts are up
      end
    end
  end
end

#STDERR.puts "Ansible Groups: #{ansible_groups.inspect}"
#STDERR.puts "Ansible Host Vars: #{ansible_hostvars.inspect}"

# Generate ansible.cfg files if they don't already exist, both in the
# top level directory, and the ansible subdirectory, with appropriate
# relative path inventory entries for their location.
[{"dir" => "", "inv_dir" => "ansible"},
 {"dir" => "ansible"}].each do |cfg_dir|

  # Generate absolute path of the ansible.cfg file to generate
  cfg_file = File.join(@dev_env_root, cfg_dir["dir"], "ansible.cfg")

  # Determine the relative path of the inventory file
  inv_path = File.basename(@dev_env_inventory)
  if cfg_dir.key?("inv_dir")
    inv_path = File.join(cfg_dir["inv_dir"], inv_path)
  end

  if !File.exists?(cfg_file)
    STDERR.puts "Generating '#{cfg_file}'."
    File.open(cfg_file, "w").write(<<-HEADER
[defaults]
# Default inventory file is none specified using -i/--inventory
inventory = #{inv_path}

# Use new method for python interpreter selection, silencing warnings
interpreter_python = auto_silent

# Fact caching
fact_caching = #{@dev_env_cfg["fact_caching"]["plugin"]}
fact_caching_prefix = #{@dev_env_cfg["fact_caching"]["prefix"]}
fact_caching_timeout = #{@dev_env_cfg["fact_caching"]["timeout"]}
fact_caching_connection = #{@dev_env_cfg["fact_caching"]["uri"]}

# Don't report skipped tasks
#stdout_callback = skippy

# Quieten retry files
retry_files_enabled = False
retry_files_save_path = /tmp/

# To help with issues becoming a non-root user use these settings.
# However see https://stackoverflow.com/questions/47873671/becoming-non-root-user-in-ansible-fails
# for some caveats
pipelining = True
allow_world_readable_tmpfiles = True

# Ref: https://docs.ansible.com/ansible/2.9/reference_appendices/config.html#default-callback-whitelist
# Show per-task profiling info and also a summary of longest
# running tasks at end of playbook run
callback_whitelist = profile_tasks

[callback_profile_tasks]
# Ref: https://docs.ansible.com/ansible/2.9/plugins/callback/profile_tasks.html
# default is 20
task_output_limit = 20
# default is descending
sort_order = descending

[inventory]
# Fail rather than continuing if inventory source fails
any_unparsed_is_failed = true

HEADER
                             )
  end
end
