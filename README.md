# Description

Vagrant lab that we use as a test bed for:

* Martini VPLS between R1, R77 and CE1 (2 tunnels each router)
* VRRP between two RouterOS routers
* IPoE over VPLS

# Diagram


![alt text](https://github.com/logingood/mikrotik-vagrant-vpls-vrrp/blob/master/diagram.png "VRRP and VPLS on virtual mikrotiks")


# Configuration


* `peer1`, `r1`, `r77` and `peer2` are connected to the network called `internet`
* `r1`, `r77` are connected to the network `core` - we use `192.168.100.0/24`
* `p1`, `p2` are connect to `core` and `core_pe` - we use `192.168.200.0/24`
* `pe1`, `pe2` are connect to `core_pe` and `ce`

Loopback uses `10.255.255.x` addresses and assigned with the iterator of the hash.
All the IP addresses can be configured by modifying the hash at the top of Vagrantfile.

We build a VPLS tunnel from `r1`, `r77` and  `pe1` (each rouer has two tunnels)

VRRP is build between interface called `internet` (bridge) of `r1` and `r77`

Loops are solved with RSTP

`peer1`, `peer2`, `ce1`, `ce2` are ubuntu18 VMs.

# Usage

Bring everything up (takes some time)

```
vagrant up
```

Destroying the lab

```
vagrant destroy -f
```

# Connect to the nodes

```
vagrant ssh $router_name
```

e.g.

```
vagrant ssh r1
```

# Important note

I spend some time trying to figure out why VPLS didn't work on Virtualbox
hypervisor. It turned out that Virtualbox interfaces must be in promiscue mode
if you are planning to use internal network to connect the routers.

Vagrant allows to do that using the following snippet

```
      r.vm.provider "virtualbox" do |virtualbox|
        virtualbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"] if h[:ip_core] || h[:ip_core_pe]
        virtualbox.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"] if h[:ip_ce] || h[:internet]
      end
```

# Box images

We are using Vagrant Box published by
[cheretbe/packer-routeros](https://github.com/cheretbe/packer-routeros)
