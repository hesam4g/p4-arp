# Implementing ARP on data plane

## Introduction

The objective of this program is to interpret ARP requests on data plane. The functionality is implemented in `arp.p4` file.

> **Note:** I have used some code from p4lang repository ([tutorials/exercises/basic/](https://github.com/p4lang/tutorials/tree/master/exercises/basic)) for implementation of the topology and running the `mininet`. I have put adequate comments for the lines of code I wrote.



## Network Topology
The topology is defined in `pod-topo/topology.json`. There are 4 hosts connected to a switch. The first host is connected to `p1` of the switch, the second host to `p2`, and so on. The following table shows the IP and MAC addresses assigned to each host.

| Host            | IP Address      | MAC Address     |
| --------------  | -------------   | -------------   | 
| Host 1          | 10.0.0.1        | 08:00:00:00:00:11|
| Host 2          | 10.0.0.2        | 08:00:00:00:00:22|
| Host 3          | 10.0.0.3        | 08:00:00:00:00:33|
| Host 4          | 10.0.0.4        | 08:00:00:00:00:44|


The tables in the data plane are filled using the `pod-topo/s1-runtime.json`. 


## ARP Header

The ARP header is defined as follow in the code.
I used [this](https://forum.p4.org/t/how-define-the-arp-header-in-the-p4-program/584/3) reference for the defination and parsing.
```
header arp_t {
    bit<16> hrd; // Hardware Type
    bit<16> pro; // Protocol Type
    bit<8> hln; // Hardware Address Length
    bit<8> pln; // Protocol Address Length
    bit<16> op;  // Opcode
    macAddr_t sha; // Sender Hardware Address
    ip4Addr_t spa; // Sender Protocol Address
    macAddr_t tha; // Target Hardware Address
    ip4Addr_t tpa; // Target Protocol Address
}
```

Moreover, I put the ARP header in the global header struct, alongside ethernet and ipv4:
```
struct headers {
   ethernet_t     ethernet;
   ipv4_t         ipv4;
   arp_t          arp;
}
```

Also, after parsing ethernet header, if ``EtherType`` indicates that the received packets is an ARP request (`EtherType: 0x806`), I parse the header as follow.
```
state parse_arp {
    packet.extract(hdr.arp);
    transition accept;
}
```



# An example of ARP request and reply in traditional networks.

For example, assume `Host 1` needs `Host 2`'s MAC address. In this case the sender is `Host 1`, and the target is `Host 2`.
So, `Host 1` broadcasts an ARP requests with following values:
```
op    = 1                     // Opcode for the ARP requests
sha   = 08:00:00:00:00:11     // Host 1 knows its mac address
spa   = 10.0.0.1              // Host 1 knows its ip address
tha   = 00:00:00:00:00:00     // Host 1 is looking for this one
tpa   = 10.0.0.2              // Host 1 specifies the target's IP
```

In a traditional network, the switch broadcasts the ARP requests. So, `Host 2` receives the ARP requests sent by `Host 1`, and detects that `Host 1` is asking for its MAC. Then, `Host 2` gets back to `Host 1` by sending an ARP reply to `Host 1`. The reply's header are filled as follow:
```
op    = 2                     // Opcode for the ARP replies
sha   = 08:00:00:00:00:22     // Host 2 knows its mac address
spa   = 10.0.0.2              // Host 2 knows its ip address
tha   = 08:00:00:00:00:11     // Host 2 gets Host 1's mac 
tpa   = 10.0.0.1              // Host 2 specifies the target's IP
```

Finally, `Host 1` receives the ARP reply, and by checking `sha` in ARP header, finds `Host 2`'s MAC.


# The overview of ARP implementation
I mentioned how ARP header is defined before.

Here, I explain how the functionality works alongside routing in the switch.

First of all, there are two tables in the `MyIngress`. One table for ARP handling, named `arp_table`. Another table, `ipv4_lpm`, is used for routing.

The program checks to see if ARP or IPV4 headers are valid. Then applies the correspond match+action table. 

Here, I only explain the first table. The latter table has only a simple action which transmits packets to proper PORT according to the destination address.

If the ARP header is valid, it means a host is looking for a MAC address.

If `hdr.arp.tpa` matches to an entry in `arp_table`, it means the required information is available for the host having `hdr.arp.tpa` IP address.

The action to response is `arp_process` which works as follow:
```
action arp_process (ip4Addr_t target_ip, macAddr_t target_mac)
{
   // Changing opcode to reply's opcode
   hdr.arp.op = 2;

   // Setting target's IP and MAC using the received information
   hdr.arp.tha = hdr.arp.sha;
   hdr.arp.tpa = hdr.arp.spa;

   // Filling the required information using the
   // data in the table
   hdr.arp.sha = target_mac;
   hdr.arp.spa = target_ip;

   // It is not mandatory, however, it would be better
   // to swap src and dst MACs 
   hdr.ethernet.srcAddr = target_mac;
   hdr.ethernet.dstAddr = hdr.arp.tha;

   // Sending back the reply to the same port.
   standard_metadata.egress_spec =  standard_metadata.ingress_port;
}
```

# Running the code

>**Note:** As I mentioned before, I am using some utility code from P4lang repo.

To run the code, navigate to project and run
```bash
make
```

It will lunch Mininet automatically.

![An example](./screenshot/1.png)



The above picture shows that 4 entries are added into the arp_table.
For instance, if `hdr.arp.tpa=10.0.0.1`, the action `arp_process` is called, and the required fields are passed to it for processing.



Also, the script creates two directories, "build" and "logs". We can check the switch logs by
```bash
tail -f ./logs/s1.log
```
![output](./screenshot/2.png)




# Checking the code
I opened two terminal assigned to `Host 1`'s netspace and one terminal assigned to `Host 2`'s by running the following line in Minineet.
```
xterm h1 h1 h2
```

In one terminal of `Host 1`, I ran `tcpdump` (the bottom-left terminal in the picture):
```bash
tcpdump -i eth0 -vvv
```

Using the same command, I ran `tcpdump` on `Host 2` (the bottom-right terminal in the picture):


In the another terminal of `Host 1`, I sent 1 ping to `Host 2`:
```bash
ping 10.0.0.2 -c 1
```
![output](./screenshot/5.png)


The `tcpdump` on  bottom-left shows that an ARP request is sent asking who has 10.0.0.2 MAC address. After that, the appropriate reply is captured. Next, the ICMP request/reply are sent. It shows that ARP managed somewhere. 

But, `Host 1`s request is not received in bottom-right terminal. So, where the request is handled?

(Another ARP request is captured in the other terminal asking for 10.0.0.1. That happens when `Host 2` wants to get back to ping and needs `Host 1`'s MAC!)


The table below shows the operations that switch does for handling arp.

![output](./screenshot/4.png)






1. In your shell, run:
   ```bash
   make run
   ```
   This will:
   * compile `basic.p4`, and
   * start the pod-topo in Mininet and configure all switches with
   the appropriate P4 program + table entries, and
   * configure all hosts with the commands listed in
   [pod-topo/topology.json](./pod-topo/topology.json)

2. You should now see a Mininet command prompt. Try to ping between
   hosts in the topology:
   ```bash
   mininet> h1 ping h2
   mininet> pingall
   ```
3. Type `exit` to leave each xterm and the Mininet command line.
   Then, to stop mininet:
   ```bash
   make stop
   ```
   And to delete all pcaps, build files, and logs:
   ```bash
   make clean
   ```

The ping failed because each switch is programmed
according to `basic.p4`, which drops all packets on arrival.
Your job is to extend this file so it forwards packets.

### A note about the control plane

A P4 program defines a packet-processing pipeline, but the rules
within each table are inserted by the control plane. When a rule
matches a packet, its action is invoked with parameters supplied by
the control plane as part of the rule.

In this exercise, we have already implemented the control plane
logic for you. As part of bringing up the Mininet instance, the
`make run` command will install packet-processing rules in the tables of
each switch. These are defined in the `sX-runtime.json` files, where
`X` corresponds to the switch number.

**Important:** We use P4Runtime to install the control plane rules. The
content of files `sX-runtime.json` refer to specific names of tables, keys, and
actions, as defined in the P4Info file produced by the compiler (look for the
file `build/basic.p4.p4info.txt` after executing `make run`). Any changes in the P4
program that add or rename tables, keys, or actions will need to be reflected in
these `sX-runtime.json` files.

## Step 2: Implement L3 forwarding

The `basic.p4` file contains a skeleton P4 program with key pieces of
logic replaced by `TODO` comments. Your implementation should follow
the structure given in this file---replace each `TODO` with logic
implementing the missing piece.

A complete `basic.p4` will contain the following components:

1. Header type definitions for Ethernet (`ethernet_t`) and IPv4 (`ipv4_t`).
2. **TODO:** Parsers for Ethernet and IPv4 that populate `ethernet_t` and `ipv4_t` fields.
3. An action to drop a packet, using `mark_to_drop()`.
4. **TODO:** An action (called `ipv4_forward`) that:
	1. Sets the egress port for the next hop.
	2. Updates the ethernet destination address with the address of the next hop.
	3. Updates the ethernet source address with the address of the switch.
	4. Decrements the TTL.
5. **TODO:** A control that:
    1. Defines a table that will read an IPv4 destination address, and
       invoke either `drop` or `ipv4_forward`.
    2. An `apply` block that applies the table.
6. **TODO:** A deparser that selects the order
    in which fields inserted into the outgoing packet.
7. A `package` instantiation supplied with the parser, control, and deparser.
    > In general, a package also requires instances of checksum verification
    > and recomputation controls. These are not necessary for this tutorial
    > and are replaced with instantiations of empty controls.

## Step 3: Run your solution

Follow the instructions from Step 1. This time, you should be able to
sucessfully ping between any two hosts in the topology.

### Food for thought

The "test suite" for your solution---sending pings between hosts in the
topology---is not very robust. What else should you test to be confident
that you implementation is correct?

> Although the Python `scapy` library is outside the scope of this tutorial,
> it can be used to generate packets for testing. The `send.py` file shows how
> to use it.

Other questions to consider:
 - How would you enhance your program to respond to ARP requests?
 - How would you enhance your program to support traceroute?
 - How would you enhance your program to support next hops?
 - Is this program enough to replace a router?  What's missing?

### Troubleshooting

There are several problems that might manifest as you develop your program:

1. `basic.p4` might fail to compile. In this case, `make run` will
report the error emitted from the compiler and halt.

2. `basic.p4` might compile but fail to support the control plane
rules in the `s1-runtime.json` through `s3-runtime.json` files that
`make run` tries to install using P4Runtime. In this case, `make run` will
report errors if control plane rules cannot be installed. Use these error
messages to fix your `basic.p4` implementation.

3. `basic.p4` might compile, and the control plane rules might be
installed, but the switch might not process packets in the desired
way. The `logs/sX.log` files contain detailed logs
that describing how each switch processes each packet. The output is
detailed and can help pinpoint logic errors in your implementation.

#### Cleaning up Mininet

In the latter two cases above, `make run` may leave a Mininet instance
running in the background. Use the following command to clean up
these instances:

```bash
make stop
```

## Relevant Documentation

The documentation for P4_16 and P4Runtime is available [here](https://p4.org/specs/)

All excercises in this repository use the v1model architecture, the documentation for which is available at:
1. The BMv2 Simple Switch target document accessible [here](https://github.com/p4lang/behavioral-model/blob/master/docs/simple_switch.md) talks mainly about the v1model architecture.
2. The include file `v1model.p4` has extensive comments and can be accessed [here](https://github.com/p4lang/p4c/blob/master/p4include/v1model.p4).