Vagrant.configure("2") do |config|
  config.vm.box = "geerlingguy/ubuntu2004"
  config.vm.box_version = "1.0.4"

  config.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: false
  config.vm.network "forwarded_port", guest: 5000, host: 5000, auto_correct: false

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yaml"
    ansible.verbose = "v"
    ansible.ask_vault_pass = true
  end
end
