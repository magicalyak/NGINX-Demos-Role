require 'fileutils'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), "vagrant/config.rb")

COREOS_URL_TEMPLATE = "https://storage.googleapis.com/%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json"

# Uniq disk UUID for libvirt
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "coreos-stable"       => {box: "coreos-stable",      user: "core", box_url: COREOS_URL_TEMPLATE % ["stable"]},
  "coreos-alpha"        => {box: "coreos-alpha",       user: "core", box_url: COREOS_URL_TEMPLATE % ["alpha"]},
  "coreos-beta"         => {box: "coreos-beta",        user: "core", box_url: COREOS_URL_TEMPLATE % ["beta"]},
  "ubuntu1604"          => {box: "generic/ubuntu1604", user: "vagrant"},
  "ubuntu1804"          => {box: "generic/ubuntu1804", user: "vagrant"},
  #"ubuntu1604"          => {box: "ubuntu/xenial64", user: "vagrant"},
  #"ubuntu1804"          => {box: "ubuntu/bionic64", user: "vagrant"},
  "centos"              => {box: "centos/7",           user: "vagrant"},
  "centos-bento"        => {box: "bento/centos-7.5",   user: "vagrant"},
  "fedora"              => {box: "fedora/28-cloud-base",                user: "vagrant"},
  "opensuse"            => {box: "opensuse/openSUSE-42.3-x86_64",       user: "vagrant"},
  "opensuse-tumbleweed" => {box: "opensuse/openSUSE-Tumbleweed-x86_64", user: "vagrant"},
}

# Defaults for config options defined in CONFIG
$demoname = "autoscaling"
$instance_name_prefix = "nginx"
$vm_gui = false
$vm_memory = 2048
$vm_cpus = 1
$shared_folders = {}
$forwarded_ports = {}
$subnet = "172.17.8"
$os = "ubuntu1804"
$override_disk_size = false
$disk_size = "20GB"
$local_path_provisioner_enabled = false
$local_path_provisioner_claim_root = "/opt/local-path-provisioner/"

$playbook = "provision.yml"

host_vars = {}

if File.exist?(CONFIG)
  require CONFIG
end

$box = SUPPORTED_OS[$os][:box]

# if $inventory is not set, try to use example
$inventory = "inventory/sample" if ! $inventory
$inventory = File.absolute_path($inventory, File.dirname(__FILE__))

# if $inventory has a hosts.ini file use it, otherwise copy over
# vars etc to where vagrant expects dynamic inventory to be
if ! File.exist?(File.join(File.dirname($inventory), "hosts.ini"))
  $vagrant_ansible = File.join(File.dirname(__FILE__), ".vagrant", "provisioners", "ansible")
  FileUtils.mkdir_p($vagrant_ansible) if ! File.exist?($vagrant_ansible)
  if ! File.exist?(File.join($vagrant_ansible,"inventory"))
    FileUtils.ln_s($inventory, File.join($vagrant_ansible,"inventory"))
  end
end


if Vagrant.has_plugin?("vagrant-proxyconf")
  $no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
  (1..$num_instances).each do |i|
      $no_proxy += ",#{$subnet}.#{i+100}"
  end
end

# Configure VM
Vagrant.configure(2) do |config|

  config.vm.box = $box
  if SUPPORTED_OS[$os].has_key? :box_url
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  if ($override_disk_size)
    unless Vagrant.has_plugin?("vagrant-disksize")
      system "vagrant plugin install vagrant-disksize"
    end
    config.disksize.size = $disk_size
  end

  config.vm.define vm_name = $instance_name_prefix + "-demovm" do |node|

    node.vm.hostname = vm_name

    if Vagrant.has_plugin?("vagrant-proxyconf")
      node.proxy.http     = ENV['HTTP_PROXY'] || ENV['http_proxy'] || ""
      node.proxy.https    = ENV['HTTPS_PROXY'] || ENV['https_proxy'] ||  ""
      node.proxy.no_proxy = $no_proxy
    end

    ["vmware_fusion", "vmware_workstation"].each do |vmware|
      node.vm.provider vmware do |v|
        v.vmx['memsize'] = $vm_memory
        v.vmx['numvcpus'] = $vm_cpus
      end
    end

    node.vm.provider :virtualbox do |vb|
      vb.memory = $vm_memory
      vb.cpus = $vm_cpus
      vb.gui = $vm_gui
      vb.linked_clone = true
      vb.customize ["modifyvm", :id, "--vram", "8"] # ubuntu defaults to 256 MB which is a waste of precious RAM
    end

    node.vm.provider :libvirt do |lv|
      lv.memory = $vm_memory
      lv.cpus = $vm_cpus
      lv.default_prefix = 'nginx'
      # Fix kernel panic on fedora 28
      if $os == "fedora"
        lv.cpu_mode = "host-passthrough"
      end
    end


    if $expose_docker_tcp
      node.vm.network "forwarded_port", guest: 2375, host: $expose_docker_tcp, auto_correct: true
    end

    $forwarded_ports.each do |guest, host|
      node.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
    end

    node.vm.synced_folder ".", "/vagrant", disabled: false, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z'] , rsync__exclude: ['.git','venv']
    #node.vm.synced_folder "./roles/common/files", "/src/NGINX-Demos/random-files", disabled: false, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z'], rsync__exclude: ['.git','venv']
    $shared_folders.each do |src, dst|
      node.vm.synced_folder src, dst, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z']
    end

    ip = "#{$subnet}.#{100}"
    node.vm.network :private_network, ip: ip

    # Disable swap for each vm
    node.vm.provision "shell", inline: "swapoff -a"

    host_vars[vm_name] = {
      "ip": ip,
      "docker_keepcache": "1",
      "download_run_once": "False",
      "download_localhost": "False",
      "local_path_provisioner_enabled": "#{$local_path_provisioner_enabled}",
      "local_path_provisioner_claim_root": "#{$local_path_provisioner_claim_root}"
    }

    # Only execute the Ansible provisioner once, when all the machines are up and ready.
    node.vm.provision "ansible" do |ansible|
      ansible.playbook = $playbook
      $ansible_inventory_path = File.join( $inventory, "hosts.ini")
      if File.exist?($ansible_inventory_path)
        ansible.inventory_path = $ansible_inventory_path
      end
      ansible.become = true
      ansible.limit = "all"
      ansible.host_key_checking = false
      ansible.raw_arguments = ["--flush-cache"]
      ansible.host_vars = host_vars
      #ansible.tags = ['download']
      ansible.groups = {
        "node" => ["#{$instance_name_prefix}-demovm"],
      }
    end

    if ($demoname)
      node.vm.provision "ansible" do |ansible|
        ansible.playbook = $demoname + "-demo.yml"
        $ansible_inventory_path = File.join( $inventory, "hosts.ini")
        if File.exist?($ansible_inventory_path)
          ansible.inventory_path = $ansible_inventory_path
        end
        ansible.become = true
        ansible.limit = "all"
        ansible.host_key_checking = false
        ansible.raw_arguments = ["--flush-cache"]
        ansible.host_vars = host_vars
        ansible.groups = {
          "node" => ["#{$instance_name_prefix}-demovm"],
        }
      end
    end

  end
end
