MACHINES = {
  :router1 => {
    :box_name => "centos/7",
    :vm_name => "router1",
    :net => [
			    {adapter: 2, auto_config: false, virtualbox__intnet: "net0"},
			    {ip: '192.168.10.1', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: "net1"},
            ]
  },
  :router2 => {
    :box_name => "centos/7",
    :vm_name => "router2",
    :net => [
          {adapter: 2, auto_config: false, virtualbox__intnet: "net0"},
          {ip: '192.168.20.1', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: "net2"},
            ]
  },
  :router3 => {
    :box_name => "centos/7",
    :vm_name => "router3",
    :net => [
          {adapter: 2, auto_config: false, virtualbox__intnet: "net0"},
          {ip: '192.168.30.1', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: "net3"},
            ]
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
      
    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
	    boxconfig[:net].each do |ipconf|
		  box.vm.network "private_network", ipconf
 	    end

        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          #ansible.limit = "all"
        end
	  
    end
  end
end
