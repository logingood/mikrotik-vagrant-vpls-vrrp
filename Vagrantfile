# -*- mode: ruby -*-
# vi: set ft=ruby :

routers = {
  peer1: {
    fwd_port: 18040,
    internet: '10.0.0.1'
  },
  peer2: {
    fwd_port: 18041,
    internet: '10.0.0.2'
  },
  r1: {
    fwd_port: 18001,
    internet: '10.0.0.3',
    ip_core: '192.168.100.11'
  },
  r77: {
    fwd_port: 18077,
    internet: '10.0.1.2',
    ip_core: '192.168.100.12'
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
    ip_ce: '10.0.0.5'
  },
  ce2: {
    fwd_port: 18052,
    ip_ce: '10.0.0.6'
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
        i = i + 1
        next
      end

      # Use ubuntu VMs as upstreams
      if router.to_s == "peer1" || router.to_s == "peer2"
        r.vm.box = "ubuntu/bionic64"
        r.vm.network :private_network, ip: h[:internet].to_s, netmask: 24, name: 'internet', virtualbox__intnet: "internet"
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
        r.vm.provision 'routeros_command', name: 'ospf_3', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:internet] + '/24').to_s}/24" if h[:internet]
        r.vm.provision 'routeros_command', name: 'ospf_4', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:ip_core] + '/24').to_s}/24" if h[:ip_core]
        r.vm.provision 'routeros_command', name: 'ospf_5', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:ip_core_pe] + '/24').to_s}/24" if h[:ip_core_pe]
        r.vm.provision 'routeros_command', name: 'ospf_6', command: "routing ospf network add area=backbone network=#{IPAddr.new(h[:ip_ce] + '/24').to_s}/24" if h[:ip_ce]

        # MPLS setup
        r.vm.provision 'routeros_command', name: 'mpls_2', command: "mpls ldp set enabled=yes lsr-id=10.255.255.#{i + 1} transport-address=10.255.255.#{i + 1}"
        r.vm.provision 'routeros_command', name: 'mpls_3', command: "mpls ldp interface add interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(2, n + i)}\"] name ]" if h[:ip_core]
        r.vm.provision 'routeros_command', name: 'mpls_3', command: "mpls ldp interface add interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(3, n + i)}\"] name ]" if h[:ip_core_pe]
      end

      # VPLS tunnels, advertise 1508 mtu to squeeze PPPPoE

      if router.to_s == 'pe1'
        r.vm.provision 'routeros_command', name: 'vpls1', command: '/interface vpls add disabled=no l2mtu=1508 mac-address=00:00:00:00:00:A1 name=ce1r1 remote-peer=10.255.255.3 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls2', command: '/interface bridge add mtu=1508 name=internet'
        r.vm.provision 'routeros_command', name: 'vpls3', command: '/interface bridge port add bridge=internet interface=ce1r1 horizon=1'
        r.vm.provision 'routeros_command', name: 'vpls4', command: "/interface bridge port add bridge=internet interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(4, n + i)}\"] name ]"
      end

      if router.to_s == 'r1'
        r.vm.provision 'routeros_command', name: 'vpls5', command: '/interface vpls add disabled=no l2mtu=1508 mac-address=00:00:00:00:00:A2 name=r1ce1 remote-peer=10.255.255.7 vpls-id=100:1'
        r.vm.provision 'routeros_command', name: 'vpls6', command: '/interface bridge add mtu=1508 name=internet'
        r.vm.provision 'routeros_command', name: 'vpls7', command: '/interface bridge port add bridge=internet interface=r1ce1 horizon=1'
        r.vm.provision 'routeros_command', name: 'vpls8', command: "/interface bridge port add bridge=internet interface=[/interface ethernet get [find mac-address=\"#{gen_mac_c(1, n + i)}\"] name ]"
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
