# vagrant-centos7
How to package a centos 7 box and test it with vagrant 


### Prerequisites:

[Vagrant](https://www.vagrantup.com/downloads.html)

[VirtualBox](https://www.virtualbox.org/wiki/Downloads)

[VirtualBox GuestAdditions ISO](http://download.virtualbox.org/virtualbox/5.0.14/VBoxGuestAdditions_5.0.14.iso)

[CENTOS 7 Minimal ISO ](http://centos.bio.lmu.de/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso) or pick a mirror from [here](http://isoredirect.centos.org/centos/7/isos/x86_64/)

#### Step 1: Install VirtualBox on your machine

Install virtualbox on your computer to be able run virtual machines
Navigate to the URL https://www.virtualbox.org/wiki/Downloads and download and install the version of your choice


#### Step 2: Create a CENTOS 7 Virtual Machine and attach Minimal ISO and install it.
Download the CENTOS minimal install ISO by choosing a [mirror](http://isoredirect.centos.org/centos/7/isos/x86_64/)
	or [directly](http://centos.bio.lmu.de/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso)
	
Create a new Virtual machine called CENTOS7 and add required CPU and memory and disks 

You will have NAT by default. Add any additional networking adapters you need and use appropriate host-only / bridged / Internal / NAT

Mount the ISO file and start installation

Note down your VM name from VirtualBox GUI. eg., CENTOS7


#### Step 3: Install some pre-requisite packages and those you need
 Configure proxy (optional) 
	Edit the file (__/etc/yum.conf__) to add these lines
	
	proxy=http://proxyserver.mydomain.com:8080
	# If you proxy server works with authentication
	proxy_username=yum-user
	proxy_password=password

Install epel-release package 

	sudo yum --enablerepo=extras install epel-release

Install wget package 

	sudo yum install -y wget

Install Development Tools package, since we need them later for VirtualBox Guest Additions to Compile and run

	sudo yum -y groupinstall "Development Tools"
	
	sudo yum -y install wget gcc kernel-devel-$(uname -r) kernel-headers-$(uname -r) dkms binutils gcc make patch libgomp glibc-headers glibc-devel gcc make patch dkms qt libgomp ntp

Update all those packages to latest version

	sudo yum -y update 
	
Reboot the machine and boot from the new kernel since the last command yum update may have updated the kernel too
		If you wish to exclude the kernel when you execute `yum update`
		
	sudo yum -y update --exclude=kernel*

#### Step 4: Create a vagrant user on the VM

	useradd vagrant
	echo vagrant:vagrant | chpasswd

	# Add vagrant user sudoers
	echo 'vagrant ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
	echo 'Defaults:vagrant !requiretty'  >> /etc/sudoers

#### Step 5: Make SSH settings on the machine

	# Add ssh keys required for vagrant to interact
	sudo mkdir -p /home/vagrant/.ssh
	sudo chmod 0700 /home/vagrant/.ssh
	sudo wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O /home/vagrant/.ssh/authorized_keys
	sudo chmod 0600 /home/vagrant/.ssh/authorized_keys
	sudo chown -R vagrant:vagrant /home/vagrant/.ssh
	sudo chown vagrant:vagrant .ssh/authorized_keys
	
Change this file `/etc/ssh/sshd_config`
	
	sudo vi /etc/ssh/sshd_config
	
Uncomment the following line, if exist else add the line
	
	AuthorizedKeysFile .ssh/authorized_keys

Verify you are able to authenticate with the vagrant private key from your host machine ssh with the vagrant private key

	ssh -i ~/.vagrant.d/insecure_private_key  vagrant@IP_ADDR_OF_YOUR_VM

If that ssh call succeeds, it is obvious that all vagrant ssh calls will succeed too after we package it.

#### Step 6: Install guest additions

Guest Additions are needed for vagrant to interact with the machine. one of the example is to mount a host folder to guest which is very useful for chef cookbook dvelopment and testing

[Download virtualBox Guest Additions](http://download.virtualbox.org/virtualbox/5.0.14/VBoxGuestAdditions_5.0.14.iso) 
	
	Mount the ISO to the Virtual Machine 
	On the CENTOS VM:
	
	sudo mkdir /media/cdrom/
	sudo mount /dev/cdrom /media/cdrom/
	cd /media/cdrom/
	sudo ./VBoxLinuxAdditions.run
	Reboot the system after the installation
	sudo reboot



#### Step 7: Prepare the VM for packaging

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

Reset the MAC Address of the NAT adapter, so it gets new MAC.

#### Step 8: Install Vagrant on your host machine, in this case MacOSX
Navigate to https://www.vagrantup.com/downloads.html on your browser
	Install the vagrant package for your host OS.

verify the install done successfully

		vagrant -v

#### Step 9: Package the box

make the __`VAGRANT_LOG`__ level as per your need from the available levels 		`debug` (verbose) `info` (normal) `warn` (quiet) `error` (very quiet)

	export VAGRANT_LOG=info

move to the directory where you need your box

	sudo mkdir ~/vagrantboxes
	cd ~/vagrantboxes

package the box based on the CENTOS VM you have created. `CENTOS7` is the VM we have created before

	vagrant package --base CENTOS7
		
This will create a file called package.box on the CWD. To override the output file

	vagrant package --base CENTOS7 --output centos7.box
		
It will take a moment to package it depends on the size of the VM

#### Step 10: Add the box to repository of boxes

	vagrant box list

	cd ~/Vagrantboxes

`vagrant box add <name> <boxfile>`

	vagrant box add centos7 ./centos7.box

If you have box with same name exist, remove any previously created box which is no longer needed.

`vagrant box remove <name>`


#### Step 11: Test the box you have created
	
Navigate to a directory to create a `Vagrantfile` and test it.

	mkdir ~/vagrant_centos7
	
	cd ~/vagrant_centos7
	
	vagrant init
	
This will create a default `Vagrantfile` on the directory `vagrant_centos7`

Edit the `Vagrantfile` to make following configurations

Use the centos7 box to use

modify the VM memory you really need

add any network you needed for the VM

Sample Vagrantfile

		# -*- mode: ruby -*-
		# vi: set ft=ruby :
		VAGRANTFILE_API_VERSION = 2
		Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  			config.vm.box = "centos7"
  			config.vm.boot_timeout = 1000
  			config.vm.hostname = 'centos7.mydomain.local'
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
		
	cd ~/vagrant_centos7
	vagrant up

If you have not set up `VAGRANT_LOG` before, you could use inline for `debug` level

	vagrant up --debug

Watch out for errors, if there are any

Once the VM you have just created is up and running, SSH in to it
		
	vagrant ssh
		
Hurray !! Your box is working !!
		
### Further steps could be learning the following

Add a block on Vagrantfile to execute a shell script on the VM

		$install_chef = <<SCRIPT
		if ! [ -d "/opt/chef" ];
		then
		    curl -L https://www.opscode.com/chef/install.sh | bash
		fi
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

Make sure cookbooks and data_bags directory exist under CWD where Vagrantfile lives 
eg., __~/vagrant_centos7__

You need to have librarian-chef installed 

	gem -v
	gem install libraian-chef --no-ri --no-rdoc
	
`--no-ri --no-rdoc` indicates that we skip installing documentation and ri which we never will use or refer.

do a `librarian-chef init` to create a Cheffile

	librarian-chef init

Modify the `Cheffile` to suit your needs.

For our `chef-solo` code block, we need to include cookbook 'nginx' on `Cheffile`.

run `librarian-chef install` to download any cookbook dependencies from `Cheffile`





	
