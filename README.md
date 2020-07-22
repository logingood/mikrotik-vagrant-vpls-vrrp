# Description

Vagrant lab that we use as a test bed for:

* Martini VPLS between R1, R77 and CE1 (2 tunnels each router)
* VRRP between two RouterOS routers
* IPoE over VPLS

# Diagram


![alt text](https://github.com/logingood/mikrotik-vagrant-vpls-vrrp/blob/master/diagram.png "VRRP and VPLS on virtual mikrotiks")


# Configuration


* `peer1`, `r1`, `r77` and `peer2` are connected to the network called `internet` - `10.0.10.0/24` and `10.0.10.254` is VRRP VIP
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

Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads).

Bring everything up (takes some time)

```
vagrant up
```

Once it is up, you can verify that everything works by connecting to `ce1` or `ce2`
```
vagrant ssh ce1
```

ping VRRP VIP address that is built over two VPLS bridges on r1 and r77.
```
ping 10.0.10.254
```

ping address `2.2.2.2` which we use to simulate internet host, it is an interface on `peer1`

```
ping 2.2.2.2
```

try to take down r1 or r77, and ping shouldn't interrupt.

```
vagrant halt r1
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
