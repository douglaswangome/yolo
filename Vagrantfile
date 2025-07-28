# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrant configuration for setting up a virtual machine.
# This block defines the virtual environment's properties and provisions.
Vagrant.configure("2") do |config|
  # Specifies the base box to use for the virtual machine.
  # This box provides a clean Ubuntu 20.04 LTS environment pre-configured by geerlingguy.
  config.vm.box = "geerlingguy/ubuntu2004"

  # Configures a private network for the virtual machine.
  # This allows the host machine to communicate with the guest VM
  # on a separate, isolated network using DHCP for automatic IP assignment.
  config.vm.network "private_network", type: :dhcp

  # Provider-specific configurations for VirtualBox.
  # This block allows tailoring the VM's resources when using VirtualBox as the provider.
  config.vm.provider "virtualbox" do |vb|
    # Sets the allocated memory for the virtual machine to 2048 MB (2 GB).
    vb.memory = "2048"
    # Assigns 2 virtual CPU cores to the virtual machine,
    # allowing for better performance for multi-threaded tasks.
    vb.cpus = 2
  end

  # Provisions the virtual machine using an inline shell script.
  # This block executes commands on the guest VM after it has been booted.
  config.vm.provision "shell", inline: <<-SHELL
    # Updates the package lists for upgrades and new package installations.
    apt-get update
    # Installs Python 3 and pip (package installer for Python) on the VM.
    # The -y flag automatically answers yes to prompts.
    apt-get install -y python3 python3-pip
  SHELL
end