Task1 Defining custom topologies

1. what is the output of "nodes" and "net"
>nodes
available nodes are: 
c0 h1 h2 h3 h4 h5 h6 h7 h8 s1 s2 s3 s4 s5 s6 s7
>net
h1 h1-eth0:s3-eth2
h2 h2-eth0:s3-eth3
h3 h3-eth0:s4-eth2
h4 h4-eth0:s4-eth3
h5 h5-eth0:s6-eth2
h6 h6-eth0:s6-eth3
h7 h7-eth0:s7-eth2
h8 h8-eth0:s7-eth3
s1 lo:  s1-eth1:s2-eth1 s1-eth2:s5-eth1
s2 lo:  s2-eth1:s1-eth1 s2-eth2:s3-eth1 s2-eth3:s4-eth1
s3 lo:  s3-eth1:s2-eth2 s3-eth2:h1-eth0 s3-eth3:h2-eth0
s4 lo:  s4-eth1:s2-eth3 s4-eth2:h3-eth0 s4-eth3:h4-eth0
s5 lo:  s5-eth1:s1-eth2 s5-eth2:s6-eth1 s5-eth3:s7-eth1
s6 lo:  s6-eth1:s5-eth2 s6-eth2:h5-eth0 s6-eth3:h6-eth0
s7 lo:  s7-eth1:s5-eth3 s7-eth2:h7-eth0 s7-eth3:h8-eth0
c0

2. What is the output of "h7 ifconfig"

>h7 ifconfig
h7-eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.7  netmask 255.0.0.0  broadcast 10.255.255.255
        inet6 fe80::dcaf:19ff:fe25:8964  prefixlen 64  scopeid 0x20<link>
        ether de:af:19:25:89:64  txqueuelen 1000  (Ethernet)
        RX packets 68  bytes 5212 (5.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11  bytes 866 (866.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

Task2: Analyze the "of_tutorial" controller
1. Controller behavior
self._handle_PacketIn (event) → self.act_like_hub (packet, packet_in) → self.resend_packet (pakket_in, of.OFPP_ALL) 
2. rtt(round trip time) latency test
h1 ping h2:
> h1 ping -c 100 h2
average: 1.209 ms
min/max: 0.812, 2.724 ms
h1 ping h8
> h1 ping -c 100 h8
average: 4.200 ms
min/max: 2.981, 6.414 ms
It could be observed that the latency of h1 ping h2 is a lot lower compared to ping h8. Looking back at the tree structure, it could be seen that there are less switches are passed between h1 and h2. This is leading to the different in the time.
In addition to that, it could be observed that the max latency always appear at the first ping since the switch have not seen this destination address before and will need to ask the SDN controller.
3. TCP bandwidth test
iperf tests the TCP bandwidth between two hosts inside the network. Returns(prints) [server, client] speeds
Performance between h1 and h2:
>iperf h1 h2
['19.3 Mbits/sec', '22.1 Mbits/sec']
Performance between h1 and h8:
>iperf h1 h8
['5.08 Mbits/sec', '5.58 Mbits/sec']
it 
It is pretty obvious that h1 and h2 have a lot larger bandwidth. TCP will increase the bandwidth as the connection between nodes allows. One of the aspect deciding that is the return time between nodes. With lower return time, it is assuming there are still more bandwidth available so it will continue increase. With lower rtt between h1 and h2, more bandwidth will be allowed.
4. traffic observation
When performing either of two pings, each of those switches experience at least one traffic or more. In of_tutorial, I added a log.debug which will print “traffic passing” and self.connection. This will allow the log to print whenever a traffic passes.

Task 3. MAC Learning Controller
1. How the code works
Every “switch” holds a dictionary that maps the mac address to a specific port. Whenever a “switch” receive a package, if it is not yet in the dictionary, the “switch” will “memorize” the source mac and the port it comes from in the dictionary. The next time same mac address is met, the switch will only forward the packet to the port that’s being “memorized”.

If we do a h1 ping h2, on the way out, each switch will memorize the mac address of h1 and corresponding port. When the reply goes from h2 to h1, on the way back, s3 knows port for MAC address of h1 so it will only send the package to that port.

2. rtt(round trip time) latency test
h1 ping h2:
> h1 ping -c 100 h2
average: 1.080 ms
min/max: 0.685, 1.561 ms
h1 ping h8
> h1 ping -c 100 h8
average: 3.079 ms
min/max: 2.148, 3.837 ms
the latency is lower this time. Although there will be some overhead in deciding and storing all those mappings, switches no longer need to push the message to all routers after it received the message once sourced from the source/destiny. This will greatly improve the performance especially when there are more steps between source and destiny nodes.
3. TCP bandwidth test
Performance between h1 and h2:
>iperf h1 h2
['83.4 Mbits/sec', '86.3 Mbits/sec']
Performance between h1 and h8:
>iperf h1 h8
['11.4 Mbits/sec', '12.5 Mbits/sec']
It is worth noting that the bandwidth varies a lot between different trials.
As the latency gets better, for the same reason as the Part 2, the maximum allowed bandwidth will be higher. In addition to that, just sending the message to one other router instead of broadcasting will save a lot of bandwidth for the “effective” data transfer.
