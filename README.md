 ```text
  ______   __    __     __   __     ______           __   __     _____     __     __     __    
/\  ___\ /\ "-./  \   /\ "-.\ \   /\  ___\         /\ "-.\ \   /\  __-.  /\ \  _ \ \   /\ \   
\ \  __\ \ \ \-./\ \  \ \ \-.  \  \ \ \____        \ \ \-.  \  \ \ \/\ \ \ \ \/ ".\ \  \ \ \  
  \ \_\    \ \_\ \ \_\  \ \_\\"\_\  \ \_____\        \ \_\\"\_\  \ \____-  \ \__/".~\_\  \ \_\ 
   \/_/     \/_/  \/_/   \/_/ \/_/   \/_____/         \/_/ \/_/   \/____/   \/_/   \/_/   \/_/                                                                                              
```       
----

This is the master repository for the FMNC routing research project instructed by the University of Notre Dame Wireless Instiute. Below is the inital abstract for the FMNC
project and the inital Scalebox code of which these repositories model:

> https://github.com/adstriegel/ScaleBox/tree/master/src/fmnc
>
> Fast Mobile Network Characterization Tool
>
> Premise:
>
> Divine using in-band techniques performance bottlenecks
> with respect to the last-mile wireless links and next-to-last
> mile broadband links.
>
>
> How It Works:
>
> The Fast Mobile Network Characterization tool builds on prior work
> at the University of Notre Dame with RIPPS to use similar 
> packet slicing / rearrangement to instrument the last and next-to-last
> mile wireless links.  The tool itself draws its inspiration from
> TCP Sting by Savage in the late 90's.  In short, we slice typical
> TCP payloads into smaller payload slices and then rearrange the first
> slice to the end to force multiple TCP acknowledgements.
>
> The end result is that we can effectively squeeze rapid fire
> acknowledgments out of unmodified TCP hosts using minimal bandwidth
> and consuming only a minimal amount of time.  
>   
> To illustrate, consider a normal TCP flow consisting of nearly full
> MSS (Maximum Segment Size) payloads approaching 1460 bytes.  Normally,
> one would send the packets in order while obeying normal TCP slow
> start congestion window growth (CWND).  
>
> *CWND = 1 packet*
> 	Send 1460 to client
>   Ack from client
>
> *CWND = 2 packets*
>	Send 1460 to client
>	Send 1460 to client
>		Ack from client for 2 data packets
>
> *CWND = 4 packets*
>	Send 1460 to client
>	Send 1460 to client
>	Send 1460 to client
>	Send 1460 to client 

-----

The hardware setup for the FMNC test system is characterized as follows:

1) Ubuntu 24.02 box acting as a router
2) Raspberry Pi 5 box acting as a client
3) Rasbeprry Pi 5 box acting as a server
4) "Dumb" networking switch connected on localcast

----
### Ubuntu 24.02 box acting as a router

The configuration for the Ubuntu box acting as a router is simply to route
all incoming packets, but do not ip-forward, instead use iptables to drop
incoming packets so the associated libpcap can capture the packets and modify them on the fly the achieve the FMNC workflow.

Here are relevant configs:

> $ *ip a*
```bash
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 04:7c:16:8a:5e:6b brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global noprefixroute enp3s0
       valid_lft forever preferred_lft forever
    inet 192.168.2.1/24 scope global enp3s0
       valid_lft forever preferred_lft forever
    inet6 fe80::67c:16ff:fe8a:5e6b/64 scope link 
       valid_lft forever preferred_lft forever
```

> $ *ip route*

```bash
default via 10.7.0.1 dev wlo1 proto dhcp src 10.7.177.106 metric 600 
default via 192.168.1.1 dev enp3s0 proto static metric 20100 
10.7.0.0/16 dev wlo1 proto kernel scope link src 10.7.177.106 metric 600 
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.100 metric 100 
192.168.2.0/24 dev enp3s0 proto kernel scope link src 192.168.2.1 
```

Following commands were used to drop the packets as they were routed in:

```bash
sudo iptables -I FORWARD -s 192.168.1.40 -d 192.168.2.2 -j DROP
sudo iptables -I FORWARD -s 192.168.2.2 -d 192.168.1.40 -j DROP
```

---
### RPI 5 box acting as a client

With the RPI 5s, there are two virtual interfaces which means the metric matters,
a lot of trouble was caused given the mismatch in default address and metric numbers
as connectivity to the internet as well as on a seperate subnet was required.

*Note the fact that the router traverses the two subnets to alow connectivity, else
weird connection issues occur routing on the same default address*


> $ *ip a*
```bash
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 2c:cf:67:03:31:ff brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::efdd:ad18:b446:34df/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

> $ *ip route*

```bash
0.0.0.0 via 192.168.1.1 dev eth0 proto static metric 700
default via 10.7.0.1 dev wlan0 proto dhcp src 10.7.99.68 metric 600
10.7.0.0/16 dev wlan0 proto kernel scope link src 10.7.99.68 metric 600
192.168.1.0/24 via 192.168.2.1 dev eth0
192.168.2.0/24 dev eth0 proto kernel scope link src 192.168.2.2
```

---
### RPI 5 box acting as a server

With the RPI 5s, there are two virtual interfaces which means the metric matters,
a lot of trouble was caused given the mismatch in default address and metric numbers
as connectivity to the internet as well as on a seperate subnet was required.

*Note the fact that the router traverses the two subnets to alow connectivity, else
weird connection issues occur routing on the same default address*


> $ *ip a*
```bash
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 2c:cf:67:04:45:14 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.40/24 brd 192.168.1.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 192.168.1.2/24 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::bdb5:92b5:e951:191d/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

> $ *ip route*

```bash
0.0.0.0 via 192.168.1.1 dev eth0 proto static metric 700
default via 10.7.0.1 dev wlan0 proto dhcp src 10.7.151.233 metric 600
10.7.0.0/16 dev wlan0 proto kernel scope link src 10.7.151.233 metric 600
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.40 metric 700
192.168.2.0/24 via 192.168.1.100 dev eth0
```

If you run traceroute between the two RPI 5's, you should see a ping to the router
first before the forwarding occurs to the appropriate destination

```bash
traceroute to 192.168.2.2 (192.168.2.2), 30 hops max, 60 byte packets
 1  192.168.1.100 (192.168.1.100)  0.324 ms  3.482 ms  3.469 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * 192.168.2.2 (192.168.2.2)  54.838 ms  54.798 ms
```
There is a reason for the delayed ACKs seen above, please review the code part
for more information.

---
### Building the code

The code is simple to build, all the necessary build functionality for general purposes is in the ***CREATE*** script. To build the necessary code, you will
need to install the functional dependencies for the universe repositories, look
into the ***CREATE*** script for what these dependencies are.

Some things in the ***CREATE*** script are for past projects, there is a CD/CI
build in there for Earthly that does work, but is not used in this project. 

**WARNING:** This script uses CMAKE to build the project and will use all the cores available on your machine to do it.

For now:

To install dependencies:

> $ ./CREATE --install-dependencies

To build the program:

> $ ./CREATE --build-project

To run the program at any time

> $ ./FMNC-{the naming of the program}

**Tip:** By passing "--run" by doing the following, you are able to run the program
immediately after building:

> $ ./CREATE --build-project --run


---
### The code

The code essentially implements a lower level kernel networking passthrough in userspace. We are forwarding the packets to their intendend destination at a latency of lower than >100 ms, and have full bandwidth to modify the packets in flight as we have access to all the appropriate headers and internals of the TCP/IP packet (theoretically UDP too, but that is not supported.)

When forwarding the packet, the code will automatically do an ARP lookup to find the forwarded mac adddress to point to that socket and forward the request. If you are
seeing duplciate packets, either the program is running twice or you have enabled
ip-forwarding on your router. It is necessary to force the kernel to drop incoming packets from both directions for this to work.

The program will take in the packets into a queue and store them along with necessary attributes and time stamps, before sending them off if the time between two packets is greater than 1ms (we are waiting 1ms to see if we have a big enough payload to slice).

As of writing this, the latter is not implemented, however the program does send the packet "out of order" in a sense to mimic a similar functionality, even if the payload data block is intact, the actual packets are guranteed by TCP to be received in order -- this is what we are testing.

---

### Todo

- Finish TCP payload slciing
- Heurestic for determining when SSL handshake is completed/can we slice SSL handshake
- Configuration file formatting and implementation (not all fields are implemented)
- Log files/data store similar to .pcap of incoming and outgoing files for analysis