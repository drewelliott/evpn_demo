# Underlay

## Configure the underlay

### Start with the fabric links

```
set / interface ethernet-1/1 admin-state enable
set / interface ethernet-1/1 subinterface 0 admin-state enable
set / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
set / interface ethernet-1/1 subinterface 0 ipv4 address 10.1.3.3/24 primary

set / interface ethernet-1/2 admin-state enable
set / interface ethernet-1/2 subinterface 0 admin-state enable
set / interface ethernet-1/2 subinterface 0 ipv4 admin-state enable
set / interface ethernet-1/2 subinterface 0 ipv4 address 10.2.3.3/24 primary
```

### Add the fabric interfaces to the default network-instance

```
set / network-instance default interface ethernet-1/1.0
set / network-instance default interface ethernet-1/2.0
```

### Test connectivity over these links

```
ping 10.1.3.1 network-instance default
ping 10.2.3.2 network-instance default
```

## Configure eBGP

### Create default "accept all" bgp policy

```
set / routing-policy policy all default-action policy-result accept
```

### Set the global bgp options

```
set / network-instance default protocols bgp admin-state enable
set / network-instance default protocols bgp autonomous-system 65103
set / network-instance default protocols bgp router-id 10.0.0.3
```

### Enable ipv4-unicast address family

```
set / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
```

### Configure the underlay group and assign the "accept all" policy for both import and export

```
set / network-instance default protocols bgp group eBGP-underlay admin-state enable
set / network-instance default protocols bgp group eBGP-underlay export-policy all
set / network-instance default protocols bgp group eBGP-underlay import-policy all
```

# now we configure the peers and assign them to the eBGP-underlay group

```
set / network-instance default protocols bgp neighbor 10.1.3.1 admin-state enable
set / network-instance default protocols bgp neighbor 10.1.3.1 peer-as 65101
set / network-instance default protocols bgp neighbor 10.1.3.1 peer-group eBGP-underlay
set / network-instance default protocols bgp neighbor 10.2.3.2 admin-state enable
set / network-instance default protocols bgp neighbor 10.2.3.2 peer-as 65102
set / network-instance default protocols bgp neighbor 10.2.3.2 peer-group eBGP-underlay
```

### Confirm the sessions are up and running and that we are getting underlay routes

```
show network-instance default protocols bgp neighbor
```

### We need to see all of the loopback addresses (10.0.0.1, 10.0.0.2, 10.0.0.4)

```
show network-instance default route-table | more
```

### Notice that our own loopback is not in the table, we need to configure that and add it to the default network-instance

```
set / interface system0 admin-state enable
set / interface system0 subinterface 0 admin-state enable
set / interface system0 subinterface 0 ipv4 admin-state enable
set / interface system0 subinterface 0 ipv4 address 10.0.0.3/32

set / network-instance default interface system0.0
```

### Check the route-table again and make sure that our loopback is showing up

```
show network-instance default route-table ipv4-unicast prefix 10.0.0.3/32
```

### Test connectivity by pinging to another loopback from our loopback as the source

```
ping -I 10.0.0.3 network-instance default 10.0.0.1
```

### At this point, we have the underlay established.

## Now we will configure the lag interface and ES/ESI facing the p1 router.

### Note that this is a static lag, the lacp lag is on peer2

```
set / interface lag1 admin-state enable
set / interface lag1 vlan-tagging true
set / interface lag1 subinterface 10 type bridged
set / interface lag1 subinterface 10 admin-state enable
set / interface lag1 subinterface 10 vlan encap single-tagged vlan-id 10
set / interface lag1 subinterface 20 type bridged
set / interface lag1 subinterface 20 admin-state enable
set / interface lag1 subinterface 20 vlan encap single-tagged vlan-id 20
set / interface lag1 lag lag-type static
set / interface lag1 lag min-links 1
set / interface lag1 lag member-speed 10G
```

### We have the lag configured, now add the lag to the port - ethernet-1/3

```
set / interface ethernet-1/3 admin-state enable
set / interface ethernet-1/3 ethernet aggregate-id lag1
```

### To be complete, log into **peer2** and check the _lacp_ lag

```
info interface lag1

show lag lag1 lacp-state
```

### Create ES and assign ESI

```
set / system network-instance protocols evpn ethernet-segments bgp-instance 1 ethernet-segment ES-p1 admin-state enable
set / system network-instance protocols evpn ethernet-segments bgp-instance 1 ethernet-segment ES-p1 esi 01:33:44:00:00:00:00:00:00:01
set / system network-instance protocols evpn ethernet-segments bgp-instance 1 ethernet-segment ES-p1 multi-homing-mode all-active
set / system network-instance protocols evpn ethernet-segments bgp-instance 1 ethernet-segment ES-p1 interface lag1
set / system network-instance protocols bgp-vpn bgp-instance 1
```

### Check the status of the lag and the ES

```
show lag brief

show lag member-statistics

show system network-instance ethernet-segments ES-p1 detail
```

### Note that we have no other ES-peer yet, this is because we have not enabled the evpn overlay yet, and pe3 doesn't have any way of knowing about the other peer without the MP-BGP evpn updates

# Overlay

## Create MAC-VRF and VXLAN

### First, create vxlan interfaces and assign vni to each, the vni in our configuration matches the vlan

```
set / tunnel-interface vxlan10 vxlan-interface 1 type bridged
set / tunnel-interface vxlan10 vxlan-interface 1 ingress vni 10
set / tunnel-interface vxlan20 vxlan-interface 1 type bridged
set / tunnel-interface vxlan20 vxlan-interface 1 ingress vni 20
```

### Now create the mac-vrf for each of the two vni, assign the lag subinterface and vxlan interface to the appropriate mac-vrf

> Note the import and export route-targets, these are defined as `target:asn:evi`

```
set / network-instance mac-vrf10 type mac-vrf
set / network-instance mac-vrf10 admin-state enable
set / network-instance mac-vrf10 interface lag1.10
set / network-instance mac-vrf10 vxlan-interface vxlan10.1
set / network-instance mac-vrf10 protocols bgp-evpn bgp-instance 1 admin-state enable
set / network-instance mac-vrf10 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan10.1
set / network-instance mac-vrf10 protocols bgp-evpn bgp-instance 1 evi 110
set / network-instance mac-vrf10 protocols bgp-evpn bgp-instance 1 ecmp 2
set / network-instance mac-vrf10 protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:110
set / network-instance mac-vrf10 protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:110

set / network-instance mac-vrf20 type mac-vrf
set / network-instance mac-vrf20 admin-state enable
set / network-instance mac-vrf20 interface lag1.20
set / network-instance mac-vrf20 vxlan-interface vxlan20.1
set / network-instance mac-vrf20 protocols bgp-evpn bgp-instance 1 admin-state enable
set / network-instance mac-vrf20 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan20.1
set / network-instance mac-vrf20 protocols bgp-evpn bgp-instance 1 evi 220
set / network-instance mac-vrf20 protocols bgp-evpn bgp-instance 1 ecmp 2
set / network-instance mac-vrf20 protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:220
set / network-instance mac-vrf20 protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:220
```

### Now check the bridge-table on each mac-vrf to see if we have learned any macs

```
show network-instance mac-vrf10 bridge-table mac-table all

show network-instance mac-vrf20 bridge-table mac-table all
```

### Check the vxlan interfaces and make sure they look correct

```
show tunnel-interface vxlan-interface brief
```

## Finally, we configure iBGP among the VTEPs - this will tie everything together and give us a functioning network

### Configure the iBGP-overlay group, notice that we are setting our local-as to 65000 and the peers are also 65000. We are also disabling ipv4-unicast and enabling only evpn address-family

```
set / network-instance default protocols bgp group iBGP-overlay export-policy all
set / network-instance default protocols bgp group iBGP-overlay import-policy all
set / network-instance default protocols bgp group iBGP-overlay peer-as 65000
set / network-instance default protocols bgp group iBGP-overlay afi-safi evpn admin-state enable
set / network-instance default protocols bgp group iBGP-overlay afi-safi ipv4-unicast admin-state disable
set / network-instance default protocols bgp group iBGP-overlay local-as as-number 65000
set / network-instance default protocols bgp group iBGP-overlay timers minimum-advertisement-interval 1

set / network-instance default protocols bgp neighbor 10.0.0.1 peer-group iBGP-overlay
set / network-instance default protocols bgp neighbor 10.0.0.1 transport local-address 10.0.0.3
set / network-instance default protocols bgp neighbor 10.0.0.2 peer-group iBGP-overlay
set / network-instance default protocols bgp neighbor 10.0.0.2 transport local-address 10.0.0.3
set / network-instance default protocols bgp neighbor 10.0.0.4 peer-group iBGP-overlay
set / network-instance default protocols bgp neighbor 10.0.0.4 transport local-address 10.0.0.3
```

# Validation

## Check the mac addresses learnt in each mac-vrf - take note of ESIs, those are how a lag appears to a remote VTEP

```
show network-instance mac-vrf10 bridge-table mac-table all

show network-instance mac-vrf20 bridge-table mac-table all
```

## Check the evpn routes received

### We did not have a chance to go through all of the different route types in the evpn address family, but see if you can figure out what each of the types is for

```
show network-instance default protocols bgp neighbor 10.0.0.1 received-routes evpn | more
```

### Verify end-to-end reachability from the p1 to the peer1 and peer2 routers

```
ping -I 10.0.0.5 network-instance default 1.1.1.1

show network-instance default route-table | more

show network-instance default protocols bgp neighbor 192.168.1.100 received-routes ipv4

show network-instance default protocols bgp neighbor 192.168.2.100 received-routes ipv4
```

### Check lag on p1

```
show lag member-statistics

show lag brief
```
