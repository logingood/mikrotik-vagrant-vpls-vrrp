# Description

Vagrant lab that we use as a test bed for:

* Martini VPLS between R1, R77 and CE1 (2 tunnels each router)
* VRRP between two RouterOS routers
* IPoE over VPLS

# Diagram


![alt text](https://github.com/logingood/mikrotik-vagrant-vpls-vrrp/blob/master/diagram.png "VRRP and VPLS on virtual mikrotiks")


# Configuration

* `r1` and `r77` run two VRRP groups over two bridges that connect correspodning VPLS tunnels.
  * VRRP1: 10.0.10.254/32 - takes 50% of traffic (`ce1` traffic)
  * VRRP2: 10.0.20.254/32 - takes 50% of traffic (`ce2` traffic)
* `r1`, `r77` are connected to the network `core` to run mpls, ldp and ospf - we use `192.168.100.0/24`
* `p1`, `p2` are connect to `core` and `core_pe` to run mpls, ldp and ospf - we use `192.168.200.0/24`
* `pe1`, `pe2` are connect to `core_pe` and `ce`
* `ce1` is an ubuntu host connected to `pe1` and bridged with VPLS1 (`100:1`) network - 10.0.10.10/24
* `ce2` is an ubuntu host connected to `pe2` and bridged with VPLS2 (`100:2`) network - 10.0.20.20/24

Loopback uses `10.255.255.x` addresses and assigned with the iterator of the hash.
All the IP addresses can be configured by modifying the hash at the top of Vagrantfile.

We build two L2 VPLS networks using `r1`, `r77`, `pe1` and `pe2`:

* `r1` maintains 2x VPLS tunnels to `r77` for 100:1 and 100:2 networks.
* `r77` maitnains 2x VPLS tunnesl to `r1` for 100:1 and 100:2 networks.
* `pe1` establishes tunnels to `r1` and `r77` with id 100:1
* `pe2` establishes tunnels to `r1` and `r77` with id 100:2


VRRP is built in the way that we can send 50% of traffic (from ce1) to R1 and another 50% to R77 (from ce2):

* interface called `internet` (bridge) of `r1` and `r77`. For the first VRRP group `r1` is a master with priority 100.
* interface called `internet2` (bridge) of `r1` and `r77`. For the second VRRP group `r77` is a master with priority 100.

VPLS brdige loops are solved with [bridge split
horizon](https://wiki.mikrotik.com/wiki/Manual:MPLSVPLS#Split_horizon_bridging)
on PE routers. RSTP could be used as well but the alternative link will remain
inactive, which would increase VRRP failover time.

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

# Netflow V9 and ntopng

For the sake of the experimetn we install [ntopng and nprobe](https://ntop.org) on `peer1` one.

You can open [http://127.0.0.1:3000](http://127.0.0.1:3000) in your browser and see the flows.

![alt text](https://github.com/logingood/mikrotik-vagrant-vpls-vrrp/blob/master/ntopng.png "Export netflow v9 from RouterOS and visualize with ntopng")

# Box images

We are using Vagrant Box published by
[cheretbe/packer-routeros](https://github.com/cheretbe/packer-routeros)
