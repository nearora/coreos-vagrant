# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_MASTER_PATH = File.join(File.dirname(__FILE__), "user-data-master")
CLOUD_CONFIG_WORKER_PATH = File.join(File.dirname(__FILE__), "user-data-worker")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$num_instances = 1
$instance_name_prefix = "core"
$update_channel = "alpha"
$image_version = "current"
$enable_serial_logging = false
$share_home = false
$vm_gui = false
$vm_memory = 1024
$vm_cpus = 1
$vb_cpuexecutioncap = 100
$shared_folders = {}
$forwarded_ports = {}
$insert_vagrant_insecure_key = false
$private_vm_network_prefix = "172.17.8"

# Defaults for SSL
$ca_key = "ca-key.pem"
$ca_cert = "ca.pem"
$apiserver_key = "apiserver-key.pem"
$apiserver_cert = "apiserver.pem"

# Miscellaneous
$basic_auth = "basic-auth.csv"

# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

if File.exist?(CONFIG)
  require CONFIG
end

# Use old vb_xxx config variables when set
def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = $insert_vagrant_insecure_key
  # forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true

  config.vm.box = "coreos-%s" % $update_channel
  if $image_version != "current"
      config.vm.box_version = $image_version
  end
  config.vm.box_url = "https://storage.googleapis.com/%s.release.core-os.net/amd64-usr/%s/coreos_production_vagrant.json" % [$update_channel, $image_version]

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "https://storage.googleapis.com/%s.release.core-os.net/amd64-usr/%s/coreos_production_vagrant_vmware_fusion.json" % [$update_channel, $image_version]
    end
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        ["vmware_fusion", "vmware_workstation"].each do |vmware|
          config.vm.provider vmware do |v, override|
            v.vmx["serial0.present"] = "TRUE"
            v.vmx["serial0.fileType"] = "file"
            v.vmx["serial0.fileName"] = serialFile
            v.vmx["serial0.tryNoRxLoss"] = "FALSE"
          end
        end

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), host_ip: "127.0.0.1", auto_correct: true
      end

      $forwarded_ports.each do |guest, host|
        config.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
      end

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        config.vm.provider vmware do |v|
          v.gui = vm_gui
          v.vmx['memsize'] = vm_memory
          v.vmx['numvcpus'] = vm_cpus
        end
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = vm_gui
        vb.memory = vm_memory
        vb.cpus = vm_cpus
        vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{$vb_cpuexecutioncap}"]
      end

      ip = $private_vm_network_prefix + ".#{i+$starting_ip_address}"
      config.vm.network :private_network, ip: ip

      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']
      $shared_folders.each_with_index do |(host_folder, guest_folder), index|
        config.vm.synced_folder host_folder.to_s, guest_folder.to_s, id: "core-share%02d" % index, nfs: true, mount_options: ['nolock,vers=3,udp']
      end

      if $share_home
        config.vm.synced_folder ENV['HOME'], ENV['HOME'], id: "home", :nfs => true, :mount_options => ['nolock,vers=3,udp']
      end

      $cloud_config_path = CLOUD_CONFIG_WORKER_PATH
      if i == 1
        $cloud_config_path = CLOUD_CONFIG_MASTER_PATH
      end

      if File.exist?($cloud_config_path)
        config.vm.provision :file, :source => "#{$cloud_config_path}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      end

      # Copy basic authentication details for default users
      config.vm.provision :file, :source => "#{$basic_auth}", :destination => "/tmp/#{$basic_auth}"
      config.vm.provision :shell, :inline => "mkdir -p /etc/kubernetes/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{$basic_auth} /etc/kubernetes/", :privileged => true

      # Copy SSL collateral
      config.vm.provision :file, :source => "ssl-certs/#{$ca_key}", :destination => "/tmp/#{$ca_key}"
      config.vm.provision :file, :source => "ssl-certs/#{$ca_cert}", :destination => "/tmp/#{$ca_cert}"
      config.vm.provision :file, :source => "ssl-certs/#{$apiserver_key}", :destination => "/tmp/#{$apiserver_key}"
      config.vm.provision :file, :source => "ssl-certs/#{$apiserver_cert}", :destination => "/tmp/#{$apiserver_cert}"
      config.vm.provision :file, :source => "ssl-certs/#{config.vm.hostname}-key.pem", :destination => "/tmp/#{config.vm.hostname}-key.pem"
      config.vm.provision :file, :source => "ssl-certs/#{config.vm.hostname}.pem", :destination => "/tmp/#{config.vm.hostname}.pem"

      config.vm.provision :shell, :inline => "mv /tmp/#{$ca_key} /etc/ssl/certs/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{$ca_cert} /etc/ssl/certs/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{$apiserver_key} /etc/ssl/certs/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{$apiserver_cert} /etc/ssl/certs/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{config.vm.hostname}-key.pem /etc/ssl/certs/", :privileged => true
      config.vm.provision :shell, :inline => "mv /tmp/#{config.vm.hostname}.pem /etc/ssl/certs/", :privileged => true

      # Install kubectl in /opt/bin
      config.vm.provision :shell, :inline => "mkdir -p /opt/bin", :privileged => true
      config.vm.provision :shell, :inline => "curl -s -o /opt/bin/kubectl -L https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl", :privileged => true
      config.vm.provision :shell, :inline => "chmod a+x /opt/bin/kubectl", :privileged => true
    end
  end
end
