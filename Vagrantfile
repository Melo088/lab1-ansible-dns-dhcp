Vagrant.configure("2") do |config|

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end


  # CONFIGURACIÓN DEL SERVIDOR (DNS/DHCP)
  config.vm.define "server" do |server|
    server.vm.box = "generic/rocky9"
    server.vm.hostname = "ns1.jmelol.lab"

    server.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", host_ip: "0.0.0.0"
    
    # Red 1: NAT (Vagrant la crea por defecto para internet)
    # Red 2: Red Interna (Donde vivirá el laboratorio)
    server.vm.network "private_network", ip: "192.168.100.10", virtualbox__intnet: "lab-red"
    
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.name = "Rocky9_Server_Lab"
    end
  end

  # CONFIGURACIÓN DEL CLIENTE
  config.vm.define "client" do |client|
    client.vm.box = "generic/rocky9"
    client.vm.hostname = "client.jmelol.lab"

    client.vm.network "forwarded_port", guest: 22, host: 2223, id: "ssh", host_ip: "0.0.0.0"
    
    # El cliente NO tiene IP fija, la pedirá por DHCP
    # necesitamos que esté en la misma 'Red Interna'
    client.vm.network "private_network", type: "dhcp", virtualbox__intnet: "lab-red"

    client.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "Rocky9_Client_Lab"
    end
  end
end
