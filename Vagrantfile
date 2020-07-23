# -*- mode: ruby -*-
# vi: set ft=ruby :

routers = {
  peer1: {
    fwd_port: 18040,
    internet: '10.0.1.3',
    internet_addr: '1.1.1.1'
  },
  peer2: {
    fwd_port: 18041,
    internet: '10.0.1.4',
    internet_addr: '2.2.2.2'
  },
  r1: {
    fwd_port: 18001,
    internet: '10.0.1.1',
    ip_core: '192.168.100.11',
    vpls_ip: '10.0.10.1/24',
    vrrp: '10.0.10.254/32'
  },
  r77: {
    fwd_port: 18077,
    internet: '10.0.1.2',
    ip_core: '192.168.100.12',
    vpls_ip: '10.0.10.77/24',
    vrrp: '10.0.10.254/32'
  },
  p1: {
    fwd_port: 18021,
    ip_core: '192.168.100.111',
    ip_core_pe: '192.168.200.11'
  },
  p2: {
    fwd_port: 18022,
    ip_core: '192.168.100.112',
    ip_core_pe: '192.168.200.12'
  },
  pe1: {
    fwd_port: 18031,
    ip_core_pe: '192.168.200.111',
    ip_ce: '192.168.210.11'
  },
  pe2: {
    fwd_port: 18032,
    ip_core_pe: '192.168.200.112',
    ip_ce: '192.168.210.12'
  },
  ce1: {
    fwd_port: 18051,
    ip_ce: '10.0.10.10',
    vrrp_gw: '10.0.10.254'
  },
  ce2: {
    fwd_port: 18052,
    ip_ce: '10.0.10.20',
    vrrp_gw: '10.0.10.254'
  }
}

require 'ipaddr'

Vagrant.configure("2") do |config|
  i=0
  routers.each do |router, h|
    config.vm.define router do |r|
      n = rand(5)
      r.vm.hostname = router.to_s
      # Use ubuntu VMs as Clients
      if router.to_s == "ce1" || router.to_s == "ce2"
        r.vm.box = "ubuntu/bionic64"
        r.vm.network :private_network, ip: h[:ip_ce].to_s, netmask: 24, name: 'ce', virtualbox__intnet: "ce"
        r.vm.provision :shell, :inline => "ip route delete default 2>&1 >/dev/null || true; ip route add default via #{h[:vrrp_gw]}"
        i = i + 1
        next
      end

      # Use ubuntu VMs as upstreams
      if router.to_s == "peer1" || router.to_s == "peer2"
        r.vm.box = "ubuntu/bionic64"
        r.vm.network :private_network, ip: h[:internet].to_s, netmask: 24, name: 'internet', virtualbox__intnet: "internet"
        r.vm.provision :shell, :inline => "ifconfig enp0s8:0 #{h[:internet_addr]}/32"
        r.vm.provision :shell, :inline => "ip route add 10.0.10.0/24 via 10.0.1.254"
        i = i + 1
        next
      end

      r.vm.box = "cheretbe/routeros"

      r.vm.provider "virtualbox" do |virtualbox|
        virtualbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"] if h[:ip_core] || h[:ip_core_pe]
        virtualbox.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"] if h[:ip_ce] || h[:internet]
      end

      r.vm.network "forwarded_port", guest: 80, host: h[:fwd_port]
      r.vm.network :private_network, auto_config: false, nic_type: "virtio", name: 'internet', virtualbox__intnet: "internet", mac: gen_mac(1, n + i) if h[:internet]
      r.vm.network :private_network, auto_config: false, nic_type: "virtio", name: 'core', virtualbox__intnet: "core", mac: gen_mac(2, n + i), mtu: 2000 if h[:ip_core]
      r.vm.network :private_network, auto_config: false, nic_type: "virtio", name: 'core_pe', virtualbox__intnet: "core_pe", mac: gen_mac(3, n + i), mtu: 2000 if h[:ip_core_pe]
      r.vm.network :private_network, auto_config: false, nic_type: "virtio", name: 'ce', virtualbox__intnet: "ce", mac: gen_mac(4, n + i ) if h[:ip_ce]

      #  provision ip addresses (default image doesn't have provisioner)
      r.vm.provision "routeros_command", name: "ip1", command: "ip address add address=#{h[:internet]}/24 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(1, n + i)}\"] name ]" if h[:internet]
      r.vm.provision "routeros_command", name: "ip2", command: "ip address add address=#{h[:ip_core]}/24 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(2, n + i)}\"] name ]" if h[:ip_core]
      r.vm.provision "routeros_command", name: "ip3", command: "ip address add address=#{h[:ip_core_pe]}/24 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(3, n + i)}\"] name ]" if h[:ip_core_pe]
      r.vm.provision "routeros_command", name: "ip4", command: "ip address add address=#{h[:ip_ce]}/24 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(4, n + i)}\"] name ]" if h[:ip_ce]

      unless (router.to_s == 'ce1' || router.to_s == 'ce2' || router.to_s == 'peer1' || router.to_s == 'peer2')
        # MTU  ACTUAL MTU > MPLS MTU > IP MTU
        #  14 + 1522 > 1500 + 4 + 4 + 4 + 14 > 1480 + 20
        # MPLS MTU with PPPoE + 8 bytes
        r.vm.provision 'routeros_command', name: 'mtu1', command: 'mpls interface set [find default=yes ] mpls-mtu=1534'
        # Ethernet MTU on Ethernet interfaces + eth + vlan tag ...
        r.vm.provision 'routeros_command', name: 'mtu2', command: "interface ethernet set [/interface ethernet get [find mac-address=\"#{gen_mac_c(2, n + i)}\"] name ] mtu=1560" if h[:ip_core]
        r.vm.provision 'routeros_command', name: 'mtu3', command: "interface ethernet set [/interface ethernet get [find mac-address=\"#{gen_mac_c(3, n + i)}\"] name ] mtu=1560" if h[:ip_core_pe]

        # Loopback provisioning
        r.vm.provision 'routeros_command', name: 'loopback_int', command: 'interface bridge add name=lobridge'
        r.vm.provision 'routeros_command', name: 'loopback_ip', command: "ip address add address=10.255.255.#{i + 1}/32 interface=lobridge"

        # OSPF areas
        r.vm.provision 'routeros_command', name: 'ospf_2', command: "routing ospf instance set distribute-default=never redistribute-connected=as-type-1 router-id=10.255.255.#{i + 1} numbers=default"
        r.vm.provision 'routeros_command', name: 'ospf_4', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:ip_core] + '/24').to_s}/24" if h[:ip_core]
        r.vm.provision 'routeros_command', name: 'ospf_5', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:ip_core_pe] + '/24').to_s}/24" if h[:ip_core_pe]

        # MPLS setup
        r.vm.provision 'routeros_command', name: 'mpls_2', command: "mpls ldp set enabled=yes lsr-id=10.255.255.#{i + 1} transport-address=10.255.255.#{i + 1}"
        r.vm.provision 'routeros_command', name: 'mpls_3', command: "mpls ldp interface add interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(2, n + i)}\"] name ]" if h[:ip_core]
        r.vm.provision 'routeros_command', name: 'mpls_3', command: "mpls ldp interface add interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(3, n + i)}\"] name ]" if h[:ip_core_pe]
      end

      if router.to_s == 'pe1'
        r.vm.provision 'routeros_command', name: 'vpls1', command: '/interface bridge add name=internet'
        # PE1 <-> R1
        r.vm.provision 'routeros_command', name: 'vpls2', command: '/interface vpls add disabled=no name=pe1r1 remote-peer=10.255.255.3 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls3', command: '/interface bridge port add bridge=internet interface=pe1r1 horizon=1'


        # PE1 <-> R77
        r.vm.provision 'routeros_command', name: 'vpls4', command: '/interface vpls add disabled=no name=pe1r77 remote-peer=10.255.255.4 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls5', command: '/interface bridge port add bridge=internet interface=pe1r77 horizon=1'

        # add port to ce1/ce2
        r.vm.provision 'routeros_command', name: 'vpls6', command: "/interface bridge port add bridge=internet interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(4, n + i)}\"] name ]"

      end

      if router.to_s == 'r1'
        r.vm.provision 'routeros_command', name: 'vpls7', command: '/interface bridge add name=internet'

        r.vm.provision 'routeros_command', name: 'vpls8', command: '/interface vpls add disabled=no name=r1r77 remote-peer=10.255.255.4 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls9', command: '/interface bridge port add bridge=internet interface=r1r77 horizon=1'

        r.vm.provision 'routeros_command', name: 'vpls10', command: '/interface vpls add disabled=no name=r1pe1 remote-peer=10.255.255.7 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls11', command: '/interface bridge port add bridge=internet interface=r1pe1 horizon=1'

        # VRRP for VPLS
        r.vm.provision 'routeros_command', name: 'vrrp1', command: "/interface vrrp add name=vrrp1 interface=internet priority=100"
        r.vm.provision 'routeros_command', name: 'vrrp2', command: "/ip address add address=#{h[:vpls_ip]} interface=internet"
        r.vm.provision 'routeros_command', name: 'vrrp3', command: "/ip address add address=#{h[:vrrp]} interface=vrrp1"
        # add default gw
        r.vm.provision 'routeros_command', name: 'vrrp4', command: "/ip route add dst-address=0.0.0.0/0 gateway=10.0.1.4"

        # VRRP for internet redundancy
        r.vm.provision 'routeros_command', name: 'vrrp5', command: "/interface vrrp add name=vrrp2 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(1, n + i)}\"] name ] priority=100"
        r.vm.provision 'routeros_command', name: 'vrrp3', command: "/ip address add address=10.0.1.254/32 interface=vrrp2"

        # Netflow

        r.vm.provision 'routeros_command', name: 'netflow1', command: "/ip traffic-flow set enabled=yes"
        r.vm.provision 'routeros_command', name: 'netflow2', command: "/ip traffic-flow target add dst-address=10.0.1.3 port=2055 version=9"
      end

      if router.to_s == 'r77'
        r.vm.provision 'routeros_command', name: 'vpls12', command: '/interface bridge add name=internet'

        r.vm.provision 'routeros_command', name: 'vpls13', command: '/interface vpls add disabled=no name=r77r1 remote-peer=10.255.255.3 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls14', command: '/interface bridge port add bridge=internet interface=r77r1 horizon=1'

        r.vm.provision 'routeros_command', name: 'vpls15', command: '/interface vpls add disabled=no name=r77pe1 remote-peer=10.255.255.7 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls16', command: '/interface bridge port add bridge=internet interface=r77pe1 horizon=1'

        # VRRP for VPLS
        r.vm.provision 'routeros_command', name: 'vrrp1', command: "/interface vrrp add name=vrrp1 interface=internet priority=50"
        r.vm.provision 'routeros_command', name: 'vrrp2', command: "/ip address add address=#{h[:vpls_ip]} interface=internet"
        r.vm.provision 'routeros_command', name: 'vrrp3', command: "/ip address add address=#{h[:vrrp]} interface=vrrp1"
        # add default gw
        r.vm.provision 'routeros_command', name: 'vrrp4', command: "/ip route add dst-address=0.0.0.0/0 gateway=10.0.1.4"

        # VRRP for internet redundancy
        r.vm.provision 'routeros_command', name: 'vrrp5', command: "/interface vrrp add name=vrrp2 interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(1, n + i)}\"] name ] priority=100"
        r.vm.provision 'routeros_command', name: 'vrrp3', command: "/ip address add address=10.0.1.254/32 interface=vrrp2"

        # Netflow
        r.vm.provision 'routeros_command', name: 'netflow1', command: "/ip traffic-flow set enabled=yes"
        r.vm.provision 'routeros_command', name: 'netflow2', command: "/ip traffic-flow target add dst-address=10.0.1.3 port=2055 version=9"
      end

      i+=1
    end
  end
end

def gen_mac(i, j)
  "0800271111" + i.to_s + (j).to_s(16).upcase
end

def gen_mac_c(i, j)
  "08:00:27:11:11:" + i.to_s + (j).to_s(16).upcase
end
