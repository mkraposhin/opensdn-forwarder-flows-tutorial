OpenSDN vRouter Forwarder flows tutorial
========================================

The tutorial shows how vRouter Agent instructs vRouter Forwarder about new
flows calculated from RIB tables stored in the operative database. It uses
very simple configuration to imitate UDP connections between 2 endpoints (
2 Linux docker containers). The tutorial shows:
- how to install flows manually;
- which steps of the interaction between vRouter Forwarder
and vRouter Agent are involved when new flows are installed;
- how to inspect flows stored in the corresponding vRouter Forwarder
table;
- how to delete flows from the vRouter Forwarder table;
- how to use fat flows.


It is assumed that a student has already completed
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/tutorial.md)
tutorial and is able to deploy the minimum configuration specified in
sections A, B and C of the aforementioned document. This configuration (
running vrouter kernel module, **opensdn-tools**, **cont1** and **cont2**
containers) will be used as a basis for a configuration used in this
tutorial.

A. Basic preparation steps
--------------------------

The basic configuration is created by applying steps **A**, **B** and **C** of
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/tutorial.md) tutorial on clean Ubuntu system. After completing these steps
the **host OS** must have:
1. **opensdn-tools** running container;
2. **cont1** and **cont2** running containers;
3. vrouter kernel module loaded in the **host OS** memory;
4. virtual interfaces **veth1** and **veth2** in the **host OS**;
5. the corresponding virtual interfaces **veth1c** and **veth2c**
inside the containers **cont1** and **cont2**.

Next, the materials (vRouter Forwarder requests) of this tutorial must be
downloaded into the proper container's folder (**opensdn-tools**). This
is accomplished by next commands inside the **host OS**:

    git clone https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial flows-rep
    tar cfz flows-rep.tgz flows-rep && sudo docker cp ./flows-rep.tgz opensdn-tools:/flows-rep.tgz && sudo docker exec -ti opensdn-tools tar xfz flows-rep.tgz

It is now expected that container **opensdn-tools** has 2 folders inside
it's root folder (/):
- tut-rep/ with the materials of [OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial) tutorial;
- flows-rep/ with the materials [OpenSDN vRouter Forwarder flows](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial) tutorial.


B. Configuration of routing information
---------------------------------------

Before configuring the routing information, vRouter Forwarder requires
the additional tuning, see section D of [OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial) tutorial. Namely, it is necessary to initialize
memory:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_hugepages_conf.xml

and to set the VRF table used in this tutorial:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_vrf.xml

Comparing the last request to the one used in section D of
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, it is seen that now a VRF table with number 1 is used and this must
be accounted for in other requests.

The succesfull completion of these commands can be verified by running commands:

    rt --dump 1 --family bridge
    vrftable --dump

The output must be similar to the provided in
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, section D.

Afterwards it's neccesary to enrich information about virtual interfaces **veth1c**
and **veth2c** in vRouter Forwarder. Templates for the corresponding requests are
contained in the files
[set_vif1_ip.xml]()
and
[set_vif2_ip.xml](). While these requests are svery similar to the similar
requests from [OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, they have 2 important differences:
1. there is a new field <vifr_vrf></vifr_vrf> field specifying the number
of a VRF table associated with this interface for unicast packets;
2. there is a new field <vifr_mcast_vrf></vifr_mcast_vrf> field specifying the number
of a VRF table associated with this interface for multicast packets;
3. and there is a new field <vifr_flags></vifr_flags> set to value 0x1 which
corresponds to the constant
[VIF_FLAG_POLICY_ENABLED](https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py)
meaning that packets leaving a guest OS via a port with this flag are
forwarded using the flow-based approach.

As in [OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, the value of <vifr_idx></vifr_idx> field must be set from the
**host OS** indices of **veth1** and **veth2** interfaces.

Finally, requests can be submitted to vRouter Forwarder module using
**vrcli** command inside **opensdn-tools** container:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_vif1_ip.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_vif2_ip.xml

The correctness of requests can be verified using the command **vif**
inside **contrail-tools** container:

    vif -l

The output of the command above must be similar to output observed in
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial with 2 exceptions:
- VRF tables IDs must be 1 instead of 0;
- interface flags must be PL3L2 instead of L3L2 because of
explicitly specified flag [VIF_FLAG_POLICY_ENABLED](https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py). The absence of this flag corresponds to the **Packet mode** in
OpenSDN UI of a virtual port.

Following the general sequence of steps from
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, the nexthops and MPLS labels are set:
- for **cont1** container (and **veth1c** virtual interface), [set_cont1_br_nh.xml]();
- for **cont2** container (and **veth2c** virtual interface), [set_cont2_br_nh.xml]();
- for both containers (multicast nexthop), [set_mcast_br_nh.xml]().

Next fields are updated in requests [set_cont1_br_nh.xml]() and [set_cont2_br_nh.xml]():
- <nhr_encap_oif_id></nhr_encap_oif_id> specifying ID of **veth1** or **veth2**
**host OS** interfaces depending on the corresponding guest OS virtual interface;
- <nhr_vrf></nhr_vrf> specifying ID (1) of the VRF table used in this tutorial;
- <nhr_encap></nhr_encap> specifying MAC addresses of **veth1** and **veth2**.

The procedure how to obtain **veth1** and **veth2** MAC addresses and how to set
them in the corresponding requests is presented in
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial.

For the L2 multicast [set_mcast_br_nh.xml]() nexthop request, only the field
<nhr_vrf></nhr_vrf> with value 1 must be added.

The requested nexthop and MPLS labels settings are applied by
executing **vrcli** inside **opensdn-tools**:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont1_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont2_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mpls1.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mpls2.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mcast_br_nh.xml

Upon the succesfull execution of all commands (without error messages), the results can be verified using **nh** command inside **opensdn-tools** contaner:

    nh --list

Finally, L2-routes are needed

The procedure of virtual interfaces creation in this tutorial is similar

Write an intro for vRouter Forwarder simple tutorial (why it is needed)
Add a ref to vr.sandesh https://github.com/OpenSDN-io/tf-vrouter/blob/master/sandesh/vr.sandesh


Prepare environment (a reference to the basic vRouter Forwarder
tutorial is needed).

Make interfaces

    sudo bash tut-rep/scripts/make-veth veth1 veth1c cont1 10.1.1.11/24


    sudo bash tut-rep/scripts/make-veth veth2 veth2c cont2 10.1.1.22/24

Bind interfaces to vRouter

    sudo bash tut-rep/scripts/make-vif veth1
    sudo bash tut-rep/scripts/make-vif veth2


Setup vrouter

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_hugepages_conf.xml

Setup VRF table (now we use table #1)

Prepare interfaces (if they were not prepared), set interface policy to FLOW!!!
VIF_FLAG_POLICY_ENABLED = 0x1, set vifr_vrf to 1

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif1_ip.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif2_ip.xml

When we update an existing flow, a proper gen id must be used!!! VR_RFLOW_VALID flag is also needed.

Probably, since we use L3 flows, we don't need L3 routes

Add L2 nexthops: cont1, cont2, MPLS labels and mcast L2 nh
We set:
 - nhr_encap_oif_id (cont1 and cont2 nh)
 - nhr_vrf (for cont1, cont2 and mcast)
 - nhr_encap (cont1 and cont2 nh)

Add L2 routes: cont1, cont2 and mcast
 - rtr_vrf_id (for cont1, coont2 and mcast)
 - rtr_mac

General tutorial plan:
   1. Basic settings
   2. Simple flow settings (UDP, port 25600 for ingress aond egress)
   3. Fat flow, (UDP, port 25600 for ingress, any port egress)
