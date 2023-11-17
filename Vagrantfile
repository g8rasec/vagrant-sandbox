BOX_IMAGE = "ubuntu/focal64"
HOSTNAME = BOX_IMAGE.split("/").first
VM_NAME = BOX_IMAGE.split("/")[1]
USERNAME = "user"
PASSWORD = "pass"
MEMORY = "8192"
CPUs = 4
DESIRED_VAGRANT_VERSION = "2.4.0"
DESIRED_VAGRANT_FILE = "vagrant_#{DESIRED_VAGRANT_VERSION}-1_amd64.deb" 
NETWORK_INTERFACE = "wlp" # If you use a wired network, change the interface to "en0"
GATEWAY_NETWORK = `ip route | awk '/default/ && $5 ~ /#{NETWORK_INTERFACE}/ {print $3}'`.strip

unless system("which virtualbox")
  puts "VirtualBox is not installed. Installing..."
  system("sudo apt-get update")
  system("sudo apt-get install virtualbox -y")
else
  puts "VirtualBox is already installed."
end

if system("which vagrant")
  installed_version = `vagrant --version`.strip.split(" ")[1]

  if installed_version < DESIRED_VAGRANT_VERSION
    puts "Vagrant is installed, but the version is older. Updating..."
    system("wget https://releases.hashicorp.com/vagrant/#{DESIRED_VAGRANT_VERSION}/#{DESIRED_VAGRANT_FILE}")
    system("sudo apt-get install -f ./#{DESIRED_VAGRANT_FILE} -y")
  else
    puts "Vagrant is already installed and up to date."
  end
else
  puts "Vagrant is not installed. Installing..."
  system("wget https://releases.hashicorp.com/vagrant/#{DESIRED_VAGRANT_VERSION}/#{DESIRED_VAGRANT_FILE}")
  system("sudo apt-get install -f ./#{DESIRED_VAGRANT_FILE} -y")
end

def detect_interface
  interface = `ip -o link show | awk -F': ' '{print $2}' | grep ^#{NETWORK_INTERFACE}`.strip
  if interface.empty?
    raise "No compatible network interface found."
  else
    interface
  end
end

Vagrant.configure("2") do |config|
  config.vm.define VM_NAME do |host|
    host.vm.box = BOX_IMAGE
    host.vm.hostname = HOSTNAME
    host.vm.network "public_network", type: "dhcp", bridge: detect_interface
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = MEMORY
    vb.cpus = CPUs

    config.vm.provision "shell", inline: <<-SHELL
      sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
      systemctl restart sshd
      sudo apt-get update && apt-get install vim wget curl net-tools htop nmap openssh-server openfortivpn -y
      sudo route del default
      sudo route add default gw #{GATEWAY_NETWORK}

      if ! id #{USERNAME} &>/dev/null; then
        sudo useradd -d /home/#{USERNAME} -s /bin/bash #{USERNAME}
        sudo usermod -aG sudo #{USERNAME}
      fi

      if [ ! -d "/home/#{USERNAME}" ]; then
          sudo mkdir /home/#{USERNAME}
          sudo chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}
          echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd
      fi

    SHELL
  end
end
