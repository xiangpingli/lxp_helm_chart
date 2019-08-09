# Diving Into Docker Bridge Networks

In terms of Docker, most of engineers have experience in working with building microservices and deploy using Docker containers into Kubernetes. Sometimes we may feel curious about the principles of the Docker networking, how do these containers communicate with others?

So, for your better understanding of Docker Networking, this blog walks you through a step-by-step tutorial, demonstrating how to use [Bridge networks](https://docs.docker.com/network/bridge/) for containers running on the same Docker daemon host. For communication among containers running on different Docker daemon hosts, it will be demonstrated using an [overlay network](https://docs.docker.com/network/overlay/) in the next blog.

## Bridge Networks on Single Node

In this tutorial, you'll create virtual Ethernet devices, they come in pairs, a so called veth pair, which you can think of as two Ethernet cards connected by a cable. Based on IP tool you can easily assign one of the pair to the container's namespace and the other somewhere else, so that we can create virtual networks using Linux bridge.

Before getting started with the step-by-step guide, suggest you to get familiar with these four important concepts about bridged networking, [this document](https://github.com/xiaopeng163/docker-k8s-lab/blob/master/docs/source/docker/bridged-network.rst) elaborates on the basic concepts.

- Docker0 Bridge
- Network Namespace
- Veth Pair
- External Communication

![](https://pek3b.qingstor.com/kubesphere-docs/png/20190723005011.png)

## Prerequisites

Linux OS (My Linux OS is Ubuntu 16.04.10)

## Hands-on Lab

### Lab 1: Connecting Containers Using Bridge

#### Create the "Containers"

Actually, the container is composed of `Namespace + Cgroups + rootfs`. So in this lab we can just create multiple Network Namespaces as isolated environment to simulate container behavior.

```bash
$ sudo ip netns add docker0
$ sudo ip netns add docker1
```

#### Create the Veth Pairs

```bash
$ sudo ip link add veth0 type veth peer name veth1
$ sudo ip link add veth2 type veth peer name veth3
```

#### Add the Veth to the Container's Network Namespace

```bash
# docker0
$ sudo ip link set veth0 netns docker0
# docker1
$ sudo ip link set veth2 netns docker1
```

#### Inspect the Ethernet Card in docker0

```bash
$ sudo ip netns exec docker0 ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth0@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 2e:08:60:7f:85:5a brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

As we can see from the output, the veth0 has been put into docker0. Similarly, we can use command `sudo ip netns exec docker1 ip addr show` to inspect the Ethernet Card in docker1 (You'll see veth0 in the output).

At the same time, when we can inspect the Ethernet Card in host using `ip addr`, you'll find that veth0 and veth2 has disappeared, which means both veth0 and veth2 have been put into the related containers.

#### Create the Bridge

```bash
# Install brctl if you not yet install it
$ sudo apt-get install bridge-utils
# Create a bridge "br0"
$ sudo brctl addbr br0
```

#### Connect the Host Half to the Bridge

Next, let's connect the host half (i.e. veth1 and veth3) to the bridge `br0` that we created in the last step.

```
$ sudo brctl addif br0 veth1
$ sudo brctl addif br0 veth3
```

Inspect the bridge `br0` using following command, it also verifies that veth1 and veth3 are hooked up with `br0`.

```bash
$ sudo brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.32a1db06d2cf	no		veth1
							        veth3
```

#### Assign IP to Veth 

In this step, we need to assign IP address to virtual ethernet cards within the related "container".

```bash
# docker0
$ sudo ip netns exec docker0 ip addr add 172.12.0.10/24 dev veth0

# docker1
$ sudo ip netns exec docker1 ip addr add 172.12.0.12/24 dev veth2
```

Then active these ethernet cards within the "containers", set them up using following command:

```bash
# docker0
$ sudo ip netns exec docker0 ip link set veth0 up

# docker1
$ sudo ip netns exec docker1 ip link set veth2 up
```

#### Connect the Host Half to the Bridge

Similarly, we need to connect the host half (i.e. veth1 and veth3) to the bridge `br0`.

```bash
$ sudo ip link set veth1 up
$ sudo ip link set veth3 up
```

Then assign IP address to bridge `br0` and set it up.

```bash
$ sudo ip addr add 172.12.0.11/24 dev br0
$ sudo ip link set br0 up
```

#### Testing the connection

To test the communication between multiple containers, we can set up listen on `br0` first.

```
$ sudo tcpdump -i br0 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:15:30.418740 IP6 fe80::30a1:dbff:fe06:d2cf > ff02::2: ICMP6, router solicitation, length 16
```

**docker0 → docker1**

Open a new SSH terminal, enter into `docker0`, then ping `docker1` to see the network traffic.

```
$ sudo ip netns exec docker0 ping -c 3 172.12.0.12
PING 172.12.0.12 (172.12.0.12) 56(84) bytes of data.
64 bytes from 172.12.0.12: icmp_seq=1 ttl=64 time=0.084 ms
64 bytes from 172.12.0.12: icmp_seq=2 ttl=64 time=0.069 ms
64 bytes from 172.12.0.12: icmp_seq=3 ttl=64 time=0.053 ms

--- 172.12.0.12 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.053/0.068/0.084/0.015 ms
```

At the same time, we can find that the network traffic pass through `br0`

```
22:20:00.408753 ARP, Request who-has 172.12.0.12 tell 172.12.0.10, length 28
22:20:00.408792 ARP, Reply 172.12.0.12 is-at 06:e1:4f:cb:b8:e0, length 28
22:20:00.408799 IP 172.12.0.10 > 172.12.0.12: ICMP echo request, id 7950, seq 1, length 64
22:20:00.408814 IP 172.12.0.12 > 172.12.0.10: ICMP echo reply, id 7950, seq 1, length 64
22:20:01.407762 IP 172.12.0.10 > 172.12.0.12: ICMP echo request, id 7950, seq 2, length 64
22:20:01.407800 IP 172.12.0.12 > 172.12.0.10: ICMP echo reply, id 7950, seq 2, length 64
22:20:02.406769 IP 172.12.0.10 > 172.12.0.12: ICMP echo request, id 7950, seq 3, length 64
22:20:02.406798 IP 172.12.0.12 > 172.12.0.10: ICMP echo reply, id 7950, seq 3, length 64
22:20:05.410740 ARP, Request who-has 172.12.0.10 tell 172.12.0.12, length 28
22:20:05.410752 ARP, Reply 172.12.0.10 is-at 2e:08:60:7f:85:5a, length 28
```

**docker1 → docker0**

Accordingly, we can get the similar output when enter into `docker1` and ping `docker0`.

```bash
$ sudo ip netns exec docker1 ping -c 3 172.12.0.10
PING 172.12.0.10 (172.12.0.10) 56(84) bytes of data.
64 bytes from 172.12.0.10: icmp_seq=1 ttl=64 time=0.048 ms
64 bytes from 172.12.0.10: icmp_seq=2 ttl=64 time=0.068 ms
64 bytes from 172.12.0.10: icmp_seq=3 ttl=64 time=0.056 ms

--- 172.12.0.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.048/0.057/0.068/0.010 ms
```

### Lab 2: Access the Container Network from the Host

Start the service in the container `docker0` and listen on port `80`.

```bash
$ sudo ip netns exec docker0 nc -lp 80
```

Open a new SSH terminal, execute `telnet` on the host, it can be connected to port 80 of `docker0`:

```bash
$ telnet 172.12.0.10 80
Trying 172.12.0.10...
Connected to 172.12.0.10.
Escape character is '^]'.
```

Then type `hello docker`, we can find this message output from the `docker0` terminal, which means the container network is accessible outside.

### Lab 3: Externally Access the Service Exposed Within the Container

First, configure DNAT rules of iptables.

```bash
$ sudo iptables -t nat -A PREROUTING  ! -i br0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.12.0.10:80
```

Then, start the service in the container `docker0` and listen on port `80`.

```bash
$ sudo ip netns exec docker0 nc -lp 80
```

Finally, open a new terminal on another node , then access the service exposed within the `docker0` from another node, verifying whether that service started in the `docker0` can be accessed externally.

> Note: These nodes need to be placed in the same LAN.

```bash
$ telnet 192.168.0.8 80
Trying 192.168.0.8...
Connected to 192.168.0.8.
Escape character is '^]'.
hello docker
```

Then type `hello docker`, we can find this message output from the `docker0` terminal, which means the service within container is exposed successfully.

### Clean Up

```bash
$ sudo ip link set br0 down
$ sudo brctl delbr br0
$ sudo ip link  del veth1
$ sudo ip link  del veth3
```






