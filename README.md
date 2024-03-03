# What is network namespace?

In Linux, a network namespace is a virtual area that separates network configurations. It assists in keeping various processes or applications apart, each with its network environment, to stop them from interfering with one another.

## We can connect network namespaces using two methods:

  1. Create two Namespaces and connect them using veth (VM)
  2. Create two Namespaces and connect them using Linux bridge (VM/web)

Let’s start…

### Create two Namespaces and connect them using veth (vm)

**Step 1:** Check the list of network namespaces and add two new name spaces

 ```bash
sudo ip netns list
```
```bash
sudo ip netns add ns1 
sudo ip netns add ns1
```

So we created two name spaces `ns1` and `ns2`.

**Step 2:** Now we can create a virtual ethernet to connect them
```bash
sudo ip link add veth1 type veth peer veth2
```

Here,
    `veth1`: This is one end of the virtual Ethernet pair.
    `veth2`: This is the other end of the virtual Ethernet pair.

**Step 3:** Connect the two pair of virtual Ethernet with two namespaces respectively
```bash
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2
```
Now, we can check name spcaes are conneted with virtual Ethernet pair using this commands,
```bash
sudo ip netns exec ns1 ip link
sudo ip netns exec ns2 ip link
```

**Step 4:** Assign IP Addresses to Each Namespace Using Their Respective Devices — Utilize veth1 for ns1 and veth2 for ns2.
```bash
sudo ip netns exec ns1 ip addr add 192.168.1.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 192.168.1.2/24 dev veth2
```

**Step 5:** Activate the IP links for each namespace.
```bash
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up
```
We can accetive the loopback as well
```bash
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns2 ip link set lo up
```

**Step 6:** Now its time to test the connection

Execute the following command to ping ns2 from ns1:
```bash
sudo ip netns exec ns1 ping 192.168.1.2
```
Execute the following command to ping ns1 from ns2:
```bash
sudo ip netns exec ns2 ping 192.168.1.1
```
Furthermore, examining the ARP table for ns1 reveals its ability to identify its neighboring device.
```bash
ubuntu@ubuntu:~$ sudo ip netns exec ns1 arp
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.1.2                      (incomplete)                              veth1
```
Hence, The connection between the two namespaces has been established seamlessly through the use of veth.

Now, 2nd Method

### Create two Namespaces and connect them using Linux bridge (VM/web)

**Step 1:** Check the list of network namespaces and add two new name spaces
```bash
sudo ip netns list

sudo ip netns add ns1 
sudo ip netns add ns1
```

So we created two name spaces ns1 and ns2.

Step 2: Establish a Linux bridge with the name `v-net`
```bash
 sudo ip link add v-net type bridge
```

now we can up the state of the `v-net`
```bash
 sudo ip link set v-net up
```
**Step 3:** Assign an IP address to the bridge interface `v-net`
```bash
sudo ip addr add 10.0.0.1/24 dev v-net
```
**Step 4:** Now, create two virtual Ethernet interfaces using the `veth type`.
```bash
sudo ip link add veth1 type veth peer name veth-1-br
sudo ip link add veth2 type veth peer name veth-2-br
```
**Step 5:** Connect the `veth1` end of the first virtual Ethernet to `ns1`, and link the `veth2` end of the second virtual Ethernet to `ns2`.
```bash
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2
```
**Step 6:** Now, link the other end of the first virtual Ethernet (veth-1-br) with the bridge (v-net), and similarly, connect the other end of the second virtual Ethernet (veth-2-br) to the same bridge (v-net).
```bash
sudo ip link set veth-1-br master v-net
sudo ip link set veth-2-br master v-net
```
We’re all set! It’s now time to assign IP addresses to the network namespaces.

**Step 7:** Assign IP Addresses to Each Namespace Using Their Respective Devices — Utilize `veth1` for `ns1` and `veth2` for `ns2`.
```bash
sudo ip netns exec ns1 ip addr add 192.168.1.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 192.168.1.2/24 dev veth2
```
**Step 8:** Activate the IP links for each namespace.
```bash
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up

sudo ip link set dev veth-1-br up
sudo ip link set dev veth-2-br up
```
We can accetive the loopback as well
```bash
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns2 ip link set lo up
```
**Step 9:** Now its time to test the connection again

Execute the following command to ping `ns2` from `ns1`:
```bash
sudo ip netns exec ns1 ping 192.168.1.2
```
Execute the following command to ping `ns1` from `ns2`:
```bash
sudo ip netns exec ns2 ping 192.168.1.1
```
Furthermore, examining the ARP table for ns1 reveals its ability to identify its neighboring device.
```bash
ubuntu@ubuntu:~$ sudo ip netns exec ns1 arp
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.1.2                      (incomplete)                              veth1
```
Also we can set **Firewall rules**:
```bash
sudo iptables --append FORWARD --in-interface v-net --jump ACCEPT
sudo iptables --append FORWARD --out-interface v-net --jump ACCEPT
```
