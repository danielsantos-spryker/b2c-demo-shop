# Settings for the Virtualbox VM
VM_IP='10.10.0.33'
VM_MEMORY='4096'
VM_CPUS='4'

# Locations of SaltStack code
SALT_DIRECTORY="./vendor/spryker/saltstack"
SALT_REPOSITORY="git@github.com:spryker/saltstack-core.git"
SALT_BRANCH="master"
PILLAR_DIRECTORY="./vendor/spryker/pillar"
PILLAR_REPOSITORY="git@github.com:spryker/pillar-core.git"
PILLAR_BRANCH="master"

# Hostnames to be managed
HOSTS=["spryker.dev", "zed.de.spryker.dev","zed.com.spryker.dev", "www.com.spryker.dev", "com.spryker.dev", "static.com.spryker.dev", "www.de.spryker.dev", "de.spryker.dev", "static.de.spryker.dev", "kibana.spryker.dev"]

# Check whether we are running UNIX or Windows-based machine
if Vagrant::Util::Platform.windows?
  HOSTS_PATH = 'c:\WINDOWS\system32\drivers\etc\hosts'
  SYNCED_FOLDER_TYPE = 'smb'
else
  HOSTS_PATH = '/etc/hosts'
  SYNCED_FOLDER_TYPE = 'nfs'
end

# Verify if salt/pillar directories are present
require 'mkmf'

if !Dir.exists?(SALT_DIRECTORY)
  if find_executable 'git'
    system "git clone #{SALT_REPOSITORY} --branch #{SALT_BRANCH} #{SALT_DIRECTORY}"
  else
    raise "ERROR: Required #{SALT_DIRECTORY} could not be found and no git executable was found to solve this problem." +
    "\n\n\033[0m"
  end
end

if !Dir.exists?(PILLAR_DIRECTORY)
  if find_executable 'git'
    system "git clone #{PILLAR_REPOSITORY} --branch #{PILLAR_BRANCH} #{PILLAR_DIRECTORY}"
  else
    raise "ERROR: Required #{PILLAR_DIRECTORY} could not be found and no git executable was found to solve this problem." +
    "\n\n\033[0m"
  end
end

Vagrant.configure(2) do |config|
  # Base box for initial setup. Latest Debian (stable) is recommended.
  # The list of available community boxes is available on: http://www.vagrantbox.es/
  # Not that the box file should have virtualbox guest additions installed, otherwise shared folders will not work
  config.vm.box = "debian76"
  config.vm.box_url = "https://github.com/jose-lpa/packer-debian_7.6.0/releases/download/1.0/packer_virtualbox-iso_virtualbox.box"
  config.vm.hostname = "spryker-vagrant"
  config.vm.boot_timeout = 300

  # Enable ssh agent forwarding
  config.ssh.forward_agent = true

  # The VirtualBox IP-address for the browser
  config.vm.network :private_network, ip: VM_IP

  # Port forwarding for services running on VM:
  config.vm.network "forwarded_port", guest: 1080,  host: 1080,  auto_correct: true   # Mailcatcher
  config.vm.network "forwarded_port", guest: 3306,  host: 3306,  auto_correct: true   # MySQL
  config.vm.network "forwarded_port", guest: 5432,  host: 5432,  auto_correct: true   # PostgreSQL
  config.vm.network "forwarded_port", guest: 9200,  host: 9200,  auto_correct: true   # ELK-Elasticsearch
  config.vm.network "forwarded_port", guest: 10007, host: 10007, auto_correct: true   # Jenkins (development)
  config.vm.network "forwarded_port", guest: 11007, host: 11007, auto_correct: true   # Jenkins (testing)

  # install required, but missing dependencies into the base box
  config.vm.provision "shell", inline: "sudo apt-get install -y pkg-config python2.7-dev"

  # SaltStack masterless setup
  if Dir.exists?(PILLAR_DIRECTORY) && Dir.exists?(SALT_DIRECTORY)
    config.vm.synced_folder SALT_DIRECTORY,   "/srv/salt/",   type: SYNCED_FOLDER_TYPE
    config.vm.synced_folder PILLAR_DIRECTORY, "/srv/pillar/", type: SYNCED_FOLDER_TYPE
    config.vm.provision :salt do |salt|
      salt.minion_config = "salt/minion"
      salt.run_highstate = true
      salt.bootstrap_options = "-P"
    end
  else
    raise "ERROR: Salt (#{SALT_DIRECTORY}) or Pillar (#{PILLAR_DIRECTORY}) directory not found.\n\n\033[0m"
  end

  # add hosts to /etc/hosts
  if Vagrant.has_plugin? 'vagrant-hostmanager'
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true
    config.hostmanager.aliases = HOSTS
  else
    hosts_line = VM_IP + " " + HOSTS.join(' ')
    if not File.open(HOSTS_PATH).each_line.any? { |line| line.chomp == hosts_line }
      puts "WARNING: Please add the following entries to your ${HOSTS_PATH} file: \n\033[0m"
      puts hosts_line
    end
  end

  # Share the application code with VM
  config.vm.synced_folder "./", "/data/shop/development/current", type: SYNCED_FOLDER_TYPE
  if SYNCED_FOLDER_TYPE == "nfs"
    config.nfs.map_uid = Process.uid
    config.nfs.map_gid = Process.gid
  end

  # Configure VirtualBox VM resources (CPU and memory)
  config.vm.provider :virtualbox do |vb|
    vb.name = "Spryker Vagrant"
    vb.customize([
      "modifyvm", :id,
      "--memory", VM_MEMORY,
      "--cpus", VM_CPUS,
    ])
  end
end
