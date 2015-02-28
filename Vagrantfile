# -*- mode: ruby -*-
# vi: set ft=ruby :

host = RbConfig::CONFIG['host_os']

# Give VM 1/4 system memory & access to all cpu cores on the host
if host =~ /darwin/
	cpus = `sysctl -n hw.ncpu`.to_i
	# sysctl returns Bytes and we need to convert to MB
	mem = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 4
elsif host =~ /linux/
	cpus = `nproc`.to_i
	# meminfo shows KB and we need to convert to MB
	mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 4
else # sorry Windows folks, I can't help you
	cpus = 2
	mem = 1024
end

# You can ask for more memory and cores when creating your Vagrant machine:
# VAGRANT_MEMORY=2048 VAGRANT_CORES=4 vagrant up
MEMORY = ENV['VAGRANT_MEMORY'] || mem
CORES = ENV['VAGRANT_CORES'] || cpus

# Enforce lower bound of memory to 1024 MB
if MEMORY.to_i < 1024
	MEMORY = 1024
end

# Network
PRIVATE_NETWORK = ENV['VAGRANT_PRIVATE_NETWORK'] || '192.168.144.120'

# Determine if we need to forward ports
FORWARD = ENV['VAGRANT_FORWARD'] || 1

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = 2

# The folder that will contain the sources and your TYPO3 installations
SYNCEDFOLDER = ENV['VAGRANT_SYNCEDFOLDER'] || '~/Projects/TYPO3/Development'

# Boot the box with the gui enabled
DEBUG = ENV['VAGRANT_DEBUG'] || false

# Throw an error if required Vagrant plugins are not installed
plugins = { 'vagrant-hostsupdater' => nil }

plugins.each do |plugin, version|
	unless Vagrant.has_plugin? plugin
		error = "The '#{plugin}' plugin is not installed! Try running:\nvagrant plugin install #{plugin}"
		error += " --plugin-version #{version}" if version
		raise error
	end
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
	config.vm.box = 'ubuntu/trusty64'
	config.vm.boot_timeout = 180
# If you have no Internet access (can not resolve *.local.typo3.org), you can use host aliases:
# 	config.hostsupdater.aliases = [
# 		'4.5.cms.local.typo3.org',
# 		'4.5.39.cms.local.typo3.org',
# 		'6.2.cms.local.typo3.org',
# 		'6.2.9.cms.local.typo3.org',
# 		'7.0.cms.local.typo3.org',
# 		'7.0.2.cms.local.typo3.org',
# 		'1.2.neos.local.typo3.org',
# 		'dev-master.neos.local.typo3.org'
# 		]

	# Network
	config.vm.network :private_network, ip: PRIVATE_NETWORK
	if FORWARD.to_i > 0
		config.vm.network :forwarded_port, guest: 80, host: 8080
		config.vm.network :forwarded_port, guest: 3306, host: 33060
		config.vm.network :forwarded_port, guest: 35729, host: 35729
	end

	# SSH
	config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'" # avoids 'stdin: is not a tty' error.
	config.ssh.forward_agent = true
	# 	config.vm.provision "shell", inline: "echo -e '#{File.read("#{Dir.home}/.ssh/id_rsa")}' > '/home/vagrant/.ssh/id_rsa'"
	# 	config.ssh.username = "root"
	# 	config.ssh.private_key_path = "phusion.key"

	# Virtualbox
	config.vm.provider :virtualbox do |vb|
		vb.gui = !!DEBUG
		vb.memory = MEMORY.to_i
		vb.customize ["modifyvm", :id, "--cpuexecutioncap", "90"]
		vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
		vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
		vb.customize ["modifyvm", :id, "--pae", "on"]
		vb.customize ["modifyvm", :id, "--cpus", CORES.to_i]
		vb.customize ["modifyvm", :id, "--ostype", "Ubuntu_64"]

		# If more cpu's are requested than are available; enable ioapic
		if CORES.to_i > cpus
			vb.customize ["modifyvm", :id, "--ioapic", "on"]
		end
	end

	# Vmware Fusion
	config.vm.provider :vmware_fusion do |v, override|
		override.vm.box = "trusty64_fusion"
		override.vm.box_url = "http://files.vagrantup.com/trusty64_vmware_fusion.box"
		v.vmx["memsize"] = MEMORY.to_i
		v.vmx["numvcpus"] = CORES.to_i
	end

	# Parallels
	config.vm.provider :parallels do |v, override|
		v.customize ["set", :id, "--memsize", MEMORY, "--cpus", CORES]
	end

	# Ansible | http://docs.ansible.com/playbooks_best_practices.html
	config.vm.provision "ansible" do |ansible|
# 		ansible.verbose = "v"
		ansible.playbook = "site.yml"
		ansible.limit = "all"
		ansible.raw_arguments = ENV['ANSIBLE_ARGS']
		ansible.extra_vars = {
			ansible_ssh_user: 'vagrant',
			hostname: 'local.typo3.org'
		}
	end

	# Setup synced folder
	if host =~ /(darwin|linux)/
		config.vm.synced_folder SYNCEDFOLDER, "/var/www",
			id: SYNCEDFOLDER,
			:nfs => true,
			:mount_options => ['vers=3,udp,noacl,nocto,nosuid,nodev,nolock,noatime,nodiratime'],
			:linux__nfs_options => ['no_root_squash']
	else
		cfg.vm.synced_folder SYNCEDFOLDER, "/var/www/"
	end

	# Disable default shared folder
	config.vm.synced_folder ".", "/vagrant", disabled: true

	# Ensure proper permissions for nfs mounts
	config.nfs.map_uid = Process.uid
	config.nfs.map_gid = Process.gid

end
