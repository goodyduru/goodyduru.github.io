---
layout: post
title: "What I Learned From Building A Network Tunnel"
date: 2024-04-15
categories: networking
---

I've come across several mentions of [Wireguard](https://www.wireguard.com/) on the internet without ever being curious about its inner workings. I've come across multiple mentions of VPN on the internet without (again) ever being curious about its inner workings. This year, I decided to be curious about it. Since Wireguard has been touted as a really simple VPN protocol and implementation, it seemed like the perfect gateway to finally understand those sorts. Now, I have this belief that in order to strongly understand a computer-related topic, you have to at least build a simple implementation of it. That is exactly what Steve and I [did](https://github.com/systemEng-Learning/simple-vpn).

This post is a summary of knowledge gained. To keep the length manageable, I assume a basic understanding of Linux and Networking concepts.

#### 1. Network Device and Network Interface
Network devices are what allow your computer to interact with other computers over the network. Since they are peripherals, your computer needs to have an interfaces to communicate with each of them. These interfaces determine the type of packets to send to these network devices. You can see the interfaces on your computer by typing `ip link show` in command line terminal. Here's mine (I changed the IP though):

```sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether c2:5e:28:a0:10:09 brd ff:ff:ff:ff:ff:ff
```
Since it's a VPS, my machine doesn't need much. The first one (lo) stands for loopback. It's an interface that sends packet back up the kernel without leaving the machine i.e a loop. The second one (eth0) stands for ethernet. This links up the kernel to the ethernet device. 

Our machine uses a routing table to determine what packets to send to our interfaces. We can see that by typing the command `ip route show table all`

```sh
default via 138.29.160.1 dev eth0 onlink 
138.29.160.0/24 dev eth0 proto kernel scope link src 138.29.160.213 
local 127.0.0.0/8 dev lo table local proto kernel scope host src 127.0.0.1 
....truncated...
```

Bottom-up, the third says all packets with destination address within 127.0.0.0-127.255.255.255 should be routed through the loopback interface (lo). The second one says all packets with destination address 138.29.160.0-138.29.160.255 should be routed through the eth0 interface. The first one says all other packets should also be routed through the eth0 interface and be sent to the gateway/router whose IP address is 138.29.160.1.

Note the ip address that comes after *src* in the last two entries of the above output. This determines the source IP address of a packet. Majority of the times, when we try to make a connection we leave it to the operating system to select an IP address. The routing table is what the OS uses to choose an IP address. Here are 2 examples on a machine with the output of the above routing table:
1. An application wants to send packets to localhost:8080. The OS will simply look up the routing table and see that the third line is applicable. The third line states that the packet's source address is to be set to 127.0.0.1. The OS sets it and sends the packet to the lo interface.
2.  An application wants to send packets to example.com (93.184.215.14), the OS will look up the table and see that the first line is applicable. The first line states that the packet needs to be sent to 138.29.160.1. The kernel now knows that it needs to send the packets to the gateway whose IP address is 138.29.160.1. At this point, no source address has been set. Sending to the gateway now makes the second line applicable which makes the OS set the packet's source address to 138.29.160.213. After this, the packet is sent to the eth0 interface.

#### 2. I can easily create a network device and interface in software
Earlier, I said that network devices are peripherals, inferring that they are hardwares. That's partially true. Linux actually provides a way to create them in [software](https://www.kernel.org/doc/Documentation/networking/tuntap.txt) as TUN/TAP devices. This allows your application to act as a network device by just reading from and writing to the interface<sup><a href="#footer-note-1">[1]</a></sup>.

The difference between a tun device and a tap device is simple; a tun device deals with IP packets aka L3 layer, a tap device deals with Ethernet frames aka L2 layer. 

The `ip tuntap` allows you to add, delete and list all the tun/tap interfaces on your computer. You can create a tun/tap interface like this:
```sh
sudo ip tuntap add mode tun user goody group goody name tun0
```

Listing them is as simple as `sudo ip tuntap list`. Here's my output:
```sh
playtun: tun
tun0: tun persist
```

Deleting an interface is done with:
```sh
sudo ip tuntap del mode tun name tun0
```

TUN/TAP interfaces can also be created programmatically. Here's an example of creating one called 'example' in Python:

```python
is_create = True # Let tun device be persistent
name = "example" 
tun = open("/dev/net/tun", "r+b", buffering=0)  # Open the clone device.
LINUX_IFF_TUN = 0x0001  # We want a tun device
LINUX_IFF_NO_PI = 0x1000  # We don't want packet information
LINUX_TUNSETIFF = (
    0x400454CA  # Create tun device if it doesn't exist.
)
flags = LINUX_IFF_TUN | LINUX_IFF_NO_PI
ifs = struct.pack("16sH22s", name.encode("utf-8"), flags, b"")
ioctl(tun, LINUX_TUNSETIFF, ifs)
if is_create:
    # If we're creating a tun, we want it to be persistent.
    LINUX_TUNSETPERSIST = 0x400454CB
    ioctl(tun, LINUX_TUNSETPERSIST, 1)
```

The above code creates a tun interface called **example**. If the interface exists already, it gets a descriptor to it. The above code also makes the interface persistent. Persistent enables the interface to exist even after the application exits. By default, interfaces created programmatically are automatically deleted once the application exits.

The interface returned is a file descriptor which enables us to read from and write to the interfaces using the I/O API. For example to read from the above tun device in Python, it'd be like this:

```python
data = tun.read(1024) 
print(f"Read {len(data)} bytes from device {name}")
```
We can see our above created interface when we list all the tun/tap devices on our machine using `ip tuntap list`. Here's mine
```sh
playtun: tun
tun0: tun persist
example: tun persist
```
We can also see details about it by using the `ip link show dev example` command. Here's what mine is:
```sh
55: example: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 500
    link/none
```
Currently, the interface is currently inactive. We can activate it by using the command `ip link set dev example up`. To see if it's up we can look at the details by again using the `ip link show dev example` command. Here's the output now:
```sh
55: example: <NO-CARRIER,POINTOPOINT,MULTICAST,NOARP,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 500
    link/none
```
Can you notice the difference between both outputs? 

As active as the interface is, no packet will be sent to it. That's because there are no IP addresses attached to the interface. How can we know that? There's a command called `ip addr` that allows us to see this info. For example, here's the one for ethernet(eth0) by typing `ip addr show dev eth0`:
```sh
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether c2:5e:28:a0:10:09 brd ff:ff:ff:ff:ff:ff
    inet 138.29.160.213/24 brd 138.29.160.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 1a01:6d00::f021:75ff:fed6:c508/64 scope global dynamic mngtmpaddr 
       valid_lft 5140sec preferred_lft 1540sec
    inet6 ce80::f19c:00ff:fe01:c508/64 scope link 
       valid_lft forever preferred_lft forever
```
Here's the one for our loopback interface(lo) by executing `ip addr show dev lo`
```sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
```

   We can use `ip addr show dev example`:
```sh
55: example: <NO-CARRIER,POINTOPOINT,MULTICAST,NOARP,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 500
    link/none 
```
Can you spot the difference? Both the ethernet and loopback interfaces have _inet_ and _inet6_ sections, but our example interface doesn't have those. 

Without any ip address, our interface is as good as being inactive. Good thing is that Linux allows us to add ip addresses to our interface. This can be done using the command `ip addr add`. Here's my `show` output after adding the ip range 192.168.4.0-192.168.4.255 to our interface using the command `ip addr add 192.168.4.1/24 dev example`:
```sh
55: example: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 500
    link/none 
    inet 192.168.4.1/24 scope global example
       valid_lft forever preferred_lft forever
    inet6 fe80::32a1:850c:c6c1:893f/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
```

Linux automatically creates an entry in the main route table, when we add an address to an interface. We can see the entry by running the `ip route show dev example` command:
```sh
192.168.4.0/24 proto kernel scope link src 192.168.4.1 linkdown
```
Now when the OS sees a packet destined for 192.168.4.0-192.168.4.255, it sets the packet's source address to 192.168.4.1 and sends it to the application reading the example interface file descriptor. The application can process the packet, drop the packet, wrap it in another packet or even pass it along to another endpoint. The application can create a packet and write it to the interface, where the kernel can take action on the packet like send it to the right application or drop it. 

Note the **linkdown** in the previous output. This signifies that there's no application reading from the interface is running. We can easily rectify that by polling the interface descriptor in an infinite loop like this
```python
while True:
    data = tun.read(1024) 
    ...do something with the packet...
```

This section shows just a small amount of what tun/tap devices allow you to do. TUN/TAP interfaces is what enables majority of VPN applications to work.

#### 3. Not all types of packets are checked
The infrastructure underpinning the internet is fairly reliable. Most of the time it works okay but occasionally, things fail. This failure can cause data corruption. Thankfully, designers of the different protocols thought about that and came up with a way to detect this corruption via [checksumming](https://en.wikipedia.org/wiki/Checksum) and [FCS](https://en.wikipedia.org/wiki/Frame_check_sequence). 

I used to think that packets formed by all the L4, L3 and L2 protocols have error detection. I was mostly right. [UDP](https://www.geeksforgeeks.org/how-checksum-computed-in-udp/), [TCP](https://www.geeksforgeeks.org/calculation-of-tcp-checksum/), [IPv4](https://www.thegeekstuff.com/2012/05/ip-header-checksum/), [Ethernet](https://gist.github.com/sora/ae7cad0295e55f2aa12c), [PPP](https://www.rfc-editor.org/rfc/rfc1662#page-18), and [WiFi](https://chromium.googlesource.com/chromiumos/third_party/hostap/+/refs/heads/stabilize-12607.5.B-master/wlantest/crc32.c) all have this. 

The only one missing is the IPv6 protocol. It has no way to detect any error. I found this quite surprising. It turns out that the reason for this was really practical. The designers wanted speedy flow of packets and decided to ditch error detection. Now you may ask, how does this affect speedy flow? IPv4 header contains a ttl field which is used as a hop count meaning the field is decremented by one everytime a packet arrives at a router. This causes the router to recalculate the checksum. This can affect performance especially when there could be lots of routers between a client and a server<sup><a href="#footer-note-2">[2]</a></sup>. The IPv6 also has a hop count field called hop limit and the packet checksum would have needed to be recalculated everytime it passes through a router. The designers figured that since L2 and L4 protocols already had error detection, there wasn't any need for another one. Practical!

UDP's checksum is optional for IPv4, but required for IPv6. Talk about delegation :-)

#### 4. IPtables is dope
Working on a vpn and a localhost tunnelling project requires rewriting IP packets and using a NAT table. This project had a deadline and so I needed something that could handle this for me. Iptables came through in a big way. [It's](https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands) a swiss-army networking tool that allows you to do a lot of cool things. Here are some of the things I learnt:
* Let say I want to send a packet whose source is set to a private IP address e.g 10.0.0.3 over the public internet from my computer. This packet won't be routed through the public Internet. The only way is to change the IP address to my device public IP address. I can do that using iptables. Here's how:
```sh
iptables -t nat -A POSTROUTING -s 10.0.0.3/32 -j MASQUERADE
```
MASQUERADE automatically chooses the ip address which is usually the public ip address of our computer. The neat thing about the above command is that any packet sent in reply to the original packet has its destination address rewritten to 10.0.0.3 by iptables.

* Normally, 2 network interfaces cannot send packets with each other. For example, when an application reading from `example` interface tries to create a packet with eth0 IP address and sends that packet by writing to `example` interface, the kernel drops that packet. This can be a pain when you want maintain an internal nat table and modify ip packets. You can allow the interfaces using iptables like this:
```sh
iptables -A FORWARD -i example -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o example -j ACCEPT
```

It was really cool to play around with the tool.

#### 5. Wireguard as VPN
Building a very simple vpn tool really gave me a deeper understanding of how Wireguard [works](https://www.wireguard.com/#conceptual-overview). Even more, I got to really understand how to make it work as a vpn. All you have to do is to set the AllowedIPs to _0.0.0.0/0, ::0_ under the [Peer] section of the client after creating the relevant wireguard interface. Your client config could be like this assuming your client wireguard ip address is 10.0.0.2 and server is 10.0.0.1 and you've setup your wireguard on both machines.

```txt
[Interface]
Address = 10.0.0.2/32
PrivateKey = MY_PRIVATE_KEY

[Peer]
PublicKey = SERVER_PUBLICKEY
Endpoint = SERVER_IP:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

In your client, you have to change the default route in your routing table. Here's how
```sh
# gateway and interface can be gotten when you 'route -n' e.g my gateway will be 138.29.160.1 and interface will be eth0
route del default
route add -host SERVER_IP gw <gateway> dev <interface>
route add default gw 10.0.0.1 wg0
```

You should not try to run this commands on your vps before configuring your server, it will stop your ssh session. Your server config can remain as is. But you'd need to enable IP forwarding and enable source address translation with iptables. Something like this:

```sh
echo 1 > /proc/sys/net/ipv4/ip_forward # Makes your machine behave like a router
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE
```

You can increase the size of the subnet in the iptables command if you intend to support a large number of clients. That's it! Pretty straightforward.

### Conclusion
Linux networking is a really vast area of study. There are lots of cool and powerful tools to use. I love the focus in the design of wireguard. The way it just focuses on packet encryption and leaves the rest like distribution of keys to the user. It's a lesson in software design.

This was my first time working with networking with Rust and it was pretty much straightforward. 

I wrote a very simple library for tunnelling [here](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel) which was subsequently used for our [simple vpn project](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-cli) and the [localhost tunnelling project](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-local).
***

<div id="footer-note-1">[1] "Interface" here is usually referred as "device" in lots of similar virtual networking documents. I choose to call them interface so I can refer to the applications directly reading and writing to them as devices.  </div>

***

<div id="footer-note-2">[2] You can see the number of routers a packet goes through between your computer and an endpoint using traceroute. </div>
