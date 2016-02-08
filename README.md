# vagrant-centos7
How to package a centos 7 box and test it with vagrant 

How to create a centos 7 Vagrant box on MacOSX

Prerequisites:

Vagrant (https://www.vagrantup.com/downloads.html)
VirtualBox (https://www.virtualbox.org/wiki/Downloads)
VirtualBox GuestAdditions (http://download.virtualbox.org/virtualbox/5.0.14/VBoxGuestAdditions_5.0.14.iso)
CENTOS ISO - Minimal (http://centos.bio.lmu.de/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso) or pick a mirror from (http://isoredirect.centos.org/centos/7/isos/x86_64/)

Step 1: Install VirtualBox on your machine
	Install virtualbox on your computer to be able run virtual machines
	Navigate to the URL https://www.virtualbox.org/wiki/Downloads and download and install the version of your choice


Step 2: Create a CENTOS 7 Virtual Machine and attach Minimal ISO and install it.
	Download the CENTOS minimal install ISO by choosing a mirror from http://isoredirect.centos.org/centos/7/isos/x86_64/
	or directly	http://centos.bio.lmu.de/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso
	Create a new Virtual machine called CENTOS7 and add required CPU and memory and disks 
	You will have NAT by default. Add any additional networking adapters you need and use appopriate host-only / bridged / Internal / NAT
	Mount the ISO file and start installation
	Note down your VM name from VirtualBox GUI. eg., CENTOS7

Step 3: Install some pre-requisite packages and those you need
# Configure proxy (optional) 
	Edit the file (/etc/yum.conf) to add a line

	yum --enablerepo=extras install epel-release
	sudo yum install -y wget
	sudo yum -y groupinstall "Development Tools"
	sudo yum -y install wget gcc kernel-devel-$(uname -r) kernel-headers-$(uname -r) dkms binutils gcc \
	make patch libgomp glibc-headers glibc-devel kernel-headers gcc make patch dkms qt libgomp ntp
	sudo yum -y update 
	Reboot the machine and boot from the new kernel since the last command yum update may have updated the kernel too
		If you wish to exclude the kernel when you execute update
		yum -y update --exclude=kernel*

Step 4: Create a vagrant user on the VM
	useradd vagrant
	echo vagrant:vagrant | chpasswd

	# Add vagrant user sudoers
	echo 'vagrant ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
	echo 'Defaults:vagrant !requiretty'  >> /etc/sudoers

Step 5: Make SSH settings on the machine

	# Add ssh keys required for vagrant to interact
	sudo mkdir -p /home/vagrant/.ssh
	sudo chmod 0700 /home/vagrant/.ssh
	sudo wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O /home/vagrant/.ssh/authorized_keys
	sudo chmod 0600 /home/vagrant/.ssh/authorized_keys
	sudo chown -R vagrant:vagrant /home/vagrant/.ssh
	sudo chown vagrant:vagrant .ssh/authorized_keys
	# Change this file
	sudo vi /etc/ssh/sshd_config

	# AuthorizedKeysFile %h/.ssh/authorized_keys
	AuthorizedKeysFile .ssh/authorized_keys

	Verify you are able to authenticate with the vagrant private key.
	from your host machine ssh with the vagrant private key
	ssh -i ~/.vagrant.d/insecure_private_key  vagrant@IP_ADDR_OF_YOUR_VM

	If that ssh call succeeds, it is obvious all vagrant ssh calls will succeed too after we package it.

Step 6: Install guest additions
	# Guest Additions are needed for vagrant to interact with the machine.
	Download virtualBox Guest Additions
	http://download.virtualbox.org/virtualbox/5.0.14/VBoxGuestAdditions_5.0.14.iso
	Mount the ISO to the Virtual Machine 
	On the CENTOS VM:
	sudo mkdir /media/cdrom/
	sudo mount /dev/cdrom /media/cdrom/
	cd /media/cdrom/
	sudo ./VBoxLinuxAdditions.run
	Reboot the system after the installation
	sudo reboot


Step 7: Prepare the VM for packaging
	Login to your CENTOS VM 
		Remove any network entries HWADDR and UUID on /etc/sysconfig/network-scripts/ifcfg-eth0

			sudo sed -i '/HWADDR/d' /etc/sysconfig/network-scripts/ifcfg-eth0
			sudo sed -i '/UUID/d' /etc/sysconfig/network-scripts/ifcfg-eth0

		Remove the file /etc/udev/rules.d/70-persistent-net.rules which had all interface MAC addresses.

			sudo rm -rf /etc/udev/rules.d/70-persistent-net.rules

		Remove any other interface config files in use under /etc/sysconfig/network-scripts/ifcfg-ethX

			sudo rm -rf /etc/sysconfig/network-scripts/ifcfg-eth1
	Shutdown the machine
		sudo shutdown -h now

	Edit the VM Settings on VirtualBox to remove the additional adapters other the default NAT adapter.
	Reset the MAC Address 

Step 8: Install Vagrant on your host machine, in this case MacOSX
	Navigate to https://www.vagrantup.com/downloads.html on your browser
	Install the vagrant package for your host OS.

	verify the install done successfully
		vagrant -v

Step 9: Package the box
	make the VAGRANT_LOG level as per your need from the available levels 
		debug (verbose) info(normal) warn (quiet) error (very quiet)
		export VAGRANT_LOG=info

	move to the directory where you need your box
		sudo mkdir ~/vagrantboxes
		cd ~/vagrantboxes

	package the box based on the CENTOS VM you have created
		vagrant package --base CENTOS7
	This will create a file called package.box on the CWD
		vagrant package --base CENTOS7 --output centos7.box
	It will take a moment to package it depends on the size of the VM

Step 10: Add the box to repository of boxes
	vagrant box list

	cd ~/Vagrantboxes

	vagrant box add centos7 ./centos7.box

Step 11: Test the box you have created
	
	Navigate to a directory to create a Vagrantfile and test it.

	mkdir ~/vagrant_centos7
	cd ~/vagrant_centos7
	vagrant init
	This will create a default Vagrantfile on the directory vagrant_centos7

	Edit the Vagrantfile to make following configurations
		(i) Use the centos7 box to use
		(ii) modify the VM memory you really need
		(iii) add any network you needed for the VM

	Sample Vagrantfile

# -*- mode: ruby -*-
# vi: set ft=ruby :
VAGRANTFILE_API_VERSION = 2
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "centos7"
  config.vm.boot_timeout = 1000
  config.vm.hostname = 'centos67.mydomain.local'
  config.vm.network "private_network", ip: "192.168.10.200", virtualbox__hostonly: true
  config.ssh.private_key_path = "~/.vagrant.d/insecure_private_key"
  config.ssh.forward_agent = true
  config.ssh.insert_key = false
  config.vm.provider :virtualbox do |vb|
   vb.customize ["modifyvm", :id, "--memory", "512"]
   vb.gui = true
  end
end

	Power up your VM
		vagrant up
	Once the VM you have just created is up and running, SSH in to it
		vagrant ssh
Further steps could be the following
	Add a block on Vagrantfile to execute a shell script on the VM

		$install_chef = <<SCRIPT
		if ! [ -d "/opt/chef" ];
		then
		    curl -L https://www.opscode.com/chef/install.sh | bash
		fi
		yum -y install ntp
		SCRIPT

		Add this line inside the right code block

		config.vm.provision :shell, :inline => $install_chef

Use vagrant to test cookbooks.

	Add a block to test cookbooks with chef using chef-solo

		  config.vm.provision 'chef_solo' do |chef|
		    chef.cookbooks_path = "cookbooks"
		    chef.data_bags_path = "data_bags"
		    chef.add_recipe 'nginx'
		  end 

		Make sure cookbooks and data_bags directory exist under CWD where Vagrantfile lives eg., ~/vagrant_centos7

		You need to have librarian-chef installed 

		do a librarian-chef init to create a Cheffile

		Modify the Cheffile to suit your needs.

		For our chef-solo code block, we need to include cookbook 'nginx' on Cheffile.

			run librarian-chef install to download any cookbook dependencies from Cheffile





	
