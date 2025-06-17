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
tutorial, it is seen that now a VRF table with number 1 is used in our
configuration and this must be accounted for in other requests.

The succesfull completion of these commands can be verified by running commands:

    rt --dump 1 --family bridge
    vrftable --dump

The output must be similar to the provided in
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial, section D.

Afterwards it's neccesary to enrich information about virtual interfaces **veth1c**
and **veth2c** in vRouter Forwarder. Templates for the corresponding requests are
contained in the files
[set_vif1_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_vif1_ip.xml)
and
[set_vif2_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_vif2_ip.xml).
While these requests are svery similar to the similar
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
- for **cont1** container (and **veth1c** virtual interface), [set_cont1_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont1_br_nh.xml);
- for **cont2** container (and **veth2c** virtual interface), [set_cont2_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont2_br_nh.xml);
- for both containers (multicast nexthop), [set_mcast_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_mcast_br_nh.xml).

Next fields are updated in requests
[set_cont1_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont1_br_nh.xml)
and
[set_cont2_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont2_br_nh.xml) compared to
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial) tutorial:
- <nhr_encap_oif_id></nhr_encap_oif_id> specifying ID of **veth1** or **veth2**
**host OS** interfaces depending on the corresponding guest OS virtual interface;
- <nhr_vrf></nhr_vrf> specifying ID (1) of the VRF table used in this tutorial;
- <nhr_encap></nhr_encap> specifying MAC addresses of **veth1** and **veth2**.

The procedure how to obtain **veth1** and **veth2** MAC addresses and how to set
them in the corresponding requests is presented in
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial.

For the L2 multicast [set_mcast_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_mcast_br_nh.xml) nexthop request, only the field
<nhr_vrf></nhr_vrf> with value 1 must be added.

The requested nexthop and MPLS labels settings are applied by
executing **vrcli** inside **opensdn-tools**:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont1_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont2_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mpls1.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mpls2.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mcast_br_nh.xml

Upon the succesfull execution of all commands (without error messages),
the results can be verified using **nh** and **mpls** commands inside
**opensdn-tools** container:

    nh --list
    mpls --dump

Finally, the L2 routes for packets forwarding are installed in vRouter
Forwarder using
[set_mcast_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_mcast_br_rt.xml),
[set_cont1_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont1_br_rt.xml),
and
[set_cont2_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_cont2_br_rt.xml)
requests. The requests are similar to those discussed in 
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
except the new field <rtr_vrf_id></rtr_vrf_id> which, as in other
mentioned earlier requests, is used to specify the ID of the VRF table
(1) with which the interfaces are associated.
In addition, it's neccessary to supply MAC addresses of **veth1c**
and **veth2c** interfaces via <rtr_mac></rtr_mac> field, see
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial for example.

Routes are created using the commands:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont1_br_rt.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_cont2_br_rt.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_mcast_br_rt.xml

The newly created routes can be checked by **rt** command inside
**opensdn-tools** container:

    rt --dump 1 --family bridge

The output must be similar to Fig. E3 of
[OpenSDN Basic vRouter Forwarder](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial)
tutorial.

C. An ordinary flow installation procedure
------------------------------------------

Whenever a packet with unknown L3/L4 header is encountered on an interface
with flow-based forwarding, OpenSDN vRouter Forwarder creates a new
flow with parameters borrowed from that header, sets action for this flow
to Hold (H) and sends this packet to vRouter Agent for analysis. The further
procedure resembles triple TCP handshake (see Fig. C-1):
- having received the unknown packet, vRouter Agent subjects it to
analysis and tries to create a forward flow (from the packet's SIP to 
the packet's DIP) and the corresponding reverse flow (from the
packet's DIP to the packet's SIP) using the RIB obtained from
Controller and the virtual network configuration;
- when both forward and reverse flows are created, vRouter Agent
sends to vRouter Forwarder a response with the action for the
forward flow and a request with the reverse flow;
- having received this information, vRouter Forwarder updates
the action of the forward flow from the Hold state to the state
specified in the response and populates it's flow table with
a new flow record using data from the reverse flow request;
- after this, vRouter Forwarder sends the index (flow_handle)
of the reverse flow back to vRouter Agent;
- vRouter Agent receives this index (flow_handle) and finalizes
the flow installation procedure.

![Fig. C1: A comparison of the flow installation procedure in OpenSDN vRouter (right) and TCP triple handshake (left)](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-C-1.png)
*Fig. C1: A comparison of the flow installation procedure in OpenSDN vRouter (right) and TCP triple handshake (left)*

This procedure can be imitated using vRouter Forwarder kernel module,
a coupled of containers and manual requests submitted to Forwarder using
**vrcli** utils from **opensdn-tools** as follows.

Firstly, let's start **nc** program inside

- container **cont1** as a UDP client:

        nc -4 -u -p 25600 10.1.1.22 25600

- container **cont2** as a UDP server:

        nc -4 -u -l 10.1.1.22 25600
        

For simplicity, it is assumed here that both server and client programs
use UDP port number 25600 for communication.

If one sends a message from **cont1** to **cont2** via the prepared UDP
connection, they will see:
- absence of incoming packets in **cont2**
- a new flow in the vRouter Forwarder table (Fig. C2) with the state
Hold (Action:H), the flow table of vRouter Forwarder can be inspected
using **flow** command of **opensdn-tools** container
        flow -l

When a flow is in the Hold state (Action:H) it means that vRouter Forwarder
is waiting for decision from vRouter Agent. There is a queue of 3 elements
associated with each Hold flow, meaning that only 3 packets can be retained
there while vRouter Agent processes the packet. If a fourth packet arrives,
then it is discarded (dropped) by vRouter Forwarder.

![Fig. C2: The new flow in Hold state after an attempt to transmit data between **cont1** and **cont2**](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-C-2.png)
*Fig. C2: The new flow in Hold state after an attempt to transmit data between **cont1** and **cont2** *

For educational purposes, we will not update the current
flow in the Hold state, but rather create a new one with proper action.

However, it is not allowed to create a new flow record when there is already
another one in the vRouter Forwarder flow table. Therefore, it's neccesary
to delete the previous record using **flow** utility in **opensdn-tools**
container (397480 is a unique index of the flow to delete, the
actual number might different on a machine with different setting for
vRouter Forwarder):

    flow -i 397480

The new forward flow can be created using **vr_flow_req**, defined in
the interface
[vr.sandesh](https://github.com/OpenSDN-io/tf-vrouter/blob/master/sandesh/vr.sandesh).
The example of a request formulated using XML language is store in
[set_direct_1to2_flow.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_direct_1to2_flow.xml) and contains next important fields:
- <fr_op></fr_op> specifies whether the flow is created/updated/delete/inspected;
- <fr_index></fr_index> specifies the index of a flow  (-1 means that it must be
computed automatically by vRouter Forwarder);
- <fr_action></fr_action> sets actions of a flow (Hold, Forward, etc);
- <fr_flags></fr_flags> sets additional flags of a flow (more details below)
and has default value 1 (active flow);
- <fr_rindex></fr_rindex> specify the index of the corresponding reverse flow
(-1 if a reverse flow is unknown);
- <fr_family></fr_family> specifies flow family (IPv4 or IPv6);
- and others.

To install a new flow for UDP packets going from 10.1.1.11:25600 to
10.1.1.22:25600 in VRF number 1 via nexthop 1 the next command is
to be executed in **opensdn-tools**:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_direct_1to2_flow.xml

This command imitates step 1 and partially step 2 of the interaction between
vRouter Forwarder and vRouter Agent (Fig. C1).

Next, we need a reverse flow to forward UDP packets back from 10.1.1.22:25600
to 10.1.1.11:25600, this flow is created using the request stored in
[set_reverse_1to2_flow.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_reverse_1to2_flow.xml). The reverse flow request is different compared
to the forward flow request in next fields:
- source IP / destination IP values are swapped;
- source port / destination port values are swapped;
- <fr_rindex></fr_rindex> is not zero anymore, but contains the index of
the forward flow (397480, for example);
- <fr_flags></fr_flags> field now has a bit corresponding to 4096, which
signals to OpenSDN vRouter Forwarder that the being installed flow has
a reverse counterpart (specified in <fr_rindex></fr_rindex>) and
 is called [VR_RFLOW_VALID](https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py).

This flow is created by invoking in **opensdn-tools** (after changing
the request fields if necessary):

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_reverse_1to2_flow.xml

If there have been no errors, we must have two flows (fig. C3), which
correspond to the state of vRouter Agent and vRouter Forwarder before
the third step of their interaction:

    flow -l

![Fig. C3: The flow table state before the third step of vRouter Agent and vRouter Forwarder interaction](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-C-3.png)
*Fig. C3: The flow table state before the third step of vRouter Agent and vRouter Forwarder interaction*

At the last step, vRouter Forwarder links the forward flow with the reverse flow
and sends back to vRouter Agent the index of the latter. We imitate this step
with the request
[set_reverse_1to2_flow_r.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_reverse_1to2_flow_r.xml).

This request contains one more important field: <fr_gen_id></fr_gen_id> which
is used as a syncrhonization tool between vRouter Agent and vRouter Forwarder.
Each time when vRouter Forwarder receives a request to update the flow record
it checks whether:
1. it's flow key (SIP/DIP/SPORT/DPORT/PROTO/K(NH)) matches values specified
in the request;
2. the value of generation ID (*gen_id* or *Gen*) of the record matches the
value specified in <fr_gen_id></fr_gen_id> field of the request.
The flow record is updated only when there is a full match for all numbers
between the current flow record and the received request.

Therefore, before running **vrcli** with the request
[set_reverse_1to2_flow_r.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/set_reverse_1to2_flow_r.xml),
it's value of
<fr_gen_id></fr_gen_id> must be updated from the output of **flow**
utility (Gen: 8 in the Fig. C3). Afterwards, **vrcli** is executed 
in **opesdn-tools** to link the forward flow with the reverse flow:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/set_direct_1to2_flow_r.xml

The resutls of the final state can be view with **flow** utility in
**opensdn-tools** container, see Fig. C4. The sign `<=>` means that
one flow is linked with another.

![Fig. C4: The flow table state after the correct estblishment of forward and reverse flows](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-C-4.png)
*Fig. C4: The flow table state after the correct estblishment of forward and reverse flows*

Now if one tries sending data between containers **cont1** and **cont2**
via **nc** program, the data will transfer between terminal windows
of the containers.

C. A fat flow installation procedure
-------------------------------------

However, if we change the ingress port of **nc** in container **cont1**,
we are to create a new pair of flows, since 6-tuple describing flow
(SIP/DIP/SPORT/DPORT/PROTO/K(NH)) changes. Therefore, we would have
to do all the previous actions again:
- the creaton of a forward flow;
- the creation of a reverse flow;
- the linkaged of the created forward and reverse flows.

These operations are repeated by vRouter Forwarder together with
vRouter Agent each time, when, for example, a client application from a VM
creates a new UDP session with a network service (e.g., DNS), leading to
additional CPU load on the hypervisor.

This problem can be solved by employing the so called fat flows: a
concept that "unites" a range of ordinary flows into single flow record.
Fat flows can be considered as a wildcard statement instead of single
values in a 6-tuple: for instance, we can say that we want a specific
action for all packets transferring between some source and destination IP,
but for all destination ports. Fat flows wildcard (or mask) can be applied
to the next flow key fields: source IP, destination IP, source port,
destination port and to protocol.

In this tutorial, the simplest fat flow is considered (source port), but
the principle is applicable to all other types or their combinations.
The next 6-tuple is used for the fat flow key for packets travelling
from **cont1** to **cont2**:
- SIP is 10.1.1.11;
- DIP is 10.1.1.22;
- SPORT is * (any egress port);
- DPORT is 25600 (only applicable for packets destined to the port 25600);
- PROTO is 17 (UDP);
- VRF ID is 1 (corresponds to K(NH) = 1).

The requests to apply the fat flow (actually the pair of fat flows) for the
proposed case are:
- [fat_flows/set_vif1_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/fat_flows/set_vif1_ip.xml);
- [fat_flows/set_vif2_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/fat_flows/set_vif2_ip.xml);
- [fat_flows/set_direct_1to2_flow.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/fat_flows/set_direct_1to2_flow.xml);
- [fat_flows/set_reverse_1to2_flow.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/fat_flows/set_reverse_1to2_flow.xml);
- [fat_flows/set_direct_1to2_flow_r.xml](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/xml_reqs/fat_flows/set_direct_1to2_flow_r.xml).

As it can be seen from this list, fat flow parameters are assigned per 
interface and per flow.

Before proceeding to modification actions of vRouter Forwarder flow table
it is recomended to clear the table using **flow** utility in **opensdn-tools**
container. If previously created flows has indices 397480 and 5964, then
we destroy them as follows:

    flow -i 397480
    flow -i 5964


Fat flows for ports are assigned using field 
<vifr_fat_flow_protocol_port></vifr_fat_flow_protocol_port>
of request vr_interface_req. According to the interface
[vr.sandesh](https://github.com/OpenSDN-io/tf-vrouter/blob/master/sandesh/vr.sandesh),
this field is a list of 32-bit signed integers.
Each integer in the list specifies
information about destination port number and protocol
encoded as follows:
- lowest 16 bits contain port number;
- bits from 17 to 24 contain protocol number;
- bits from 25 to 32 contain other auxilliary information.

For example, if we want to encode port number 25600 for UDP protocol,
we need to put number `25600 + 17*65536` into this list.

In order to apply changes, it is necessary to run **vrcli**
inside **opensdn-tools** container:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/fat_flows/set_vif1_ip.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/fat_flows/set_vif2_ip.xml

The results of the requests execution can be verified using
**vif** utility:

    vif -l

Fig. D1 shows the modified state of the virtual interfaces **veth1c** and
**veth2c**: it is seen that now both interfaces are associated with destination
port fat flow 25600 for UDP protocol (17).

![Fig. D1: The new state of virtual interfaces after applying fat flow rules to them](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-D-1.png)
*Fig. D1: The new state of virtual interfaces after applying fat flow rules to them*

Now fat flows records can be created. Comparing to ordinary flows requests,
the requests for fat flows contain value 0 in the destination port field in
case of the forward fat flow and value 0 in the source port field in case
of the reverse fat flow.

At first forward and reverse flows are created using **vrcli** inside
**opensdn-tools** container:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/fat_flows/set_direct_1to2_flow.xml
    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/fat_flows/set_reverse_1to2_flow.xml

Then the flow table state can be expected using **flow** command
inside **opensdn-tools** container:

    flow -l

The output of the command shows (Fig. D2) that now flows have 0 as source
or destination ports (depending on a flow direction), indicating that
these are actually fat flows. We can also take value of Generation ID (Gen)
for the forward flow (5 on the Fig. D2) to set it in our final request,
which links the forward and reverse flows:

    vrcli --vr_kmode --send_sandesh_req flows-rep/xml_reqs/fat_flows/set_direct_1to2_flow_r.xml

![Fig. D2: The intermediate state of the flow table after installing partially the fat flow pair](https://github.com/mkraposhin/opensdn-forwarder-flows-tutorial/blob/main/figs/Fig-D-2.png)
*Fig. D2: The intermediate state of the flow table after installing partially the fat flow pair*

If the execution of the last request was correct, then both fat flows will be linked with
each other (having `<=>` sign in the index column).

Running **nc** program in:
- in container **cont1** as a UDP client:

        nc -4 -u 10.1.1.22 25600

- and in container **cont2** as a UDP server:

        nc -4 -u -l 10.1.1.22 25600

must yield a working UDP session without creation of additional flows.
