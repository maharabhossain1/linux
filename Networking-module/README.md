# Creating Two name spaces and connect them using Bridge and ping public ip from them

Let say we need to create two name spaces name Red and Green and bridge name is v-net

Let’s start…

### Create two Namespaces and connect them using Linux bridge (VM/web)

**Step 1:** Check the list of network namespaces and add two new name spaces

```bash
sudo ip netns list

sudo ip netns add red
sudo ip netns add green
```

So we created two name spaces red and green.

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
sudo ip addr add 192.168.1.15/24 dev v-net
```

**Step 4:** Now, create two virtual Ethernet interfaces using the `veth type`.

```bash
sudo ip link add veth-red type veth peer name veth-red-br
sudo ip link add veth-green type veth peer name veth-green-br
```

**Step 5:** Connect the `veth-red` end of the first virtual Ethernet to `red`, and link the `veth-green` end of the second virtual Ethernet to `green`.

```bash
sudo ip link set veth-red netns red
sudo ip link set veth-green netns green
```

**Step 6:** Now, link the other end of the first virtual Ethernet (veth-red-br) with the bridge (v-net), and similarly, connect the other end of the second virtual Ethernet (veth-green-br) to the same bridge (v-net).

```bash
sudo ip link set veth-red-br master v-net
sudo ip link set veth-green-br master v-net
```

We’re all set! It’s now time to assign IP addresses to the network namespaces.

**Step 7:** Assign IP Addresses to Each Namespace Using Their Respective Devices — Utilize `veth1` for `ns1` and `veth2` for `ns2`.

```bash
sudo ip netns exec red ip addr add 192.168.1.1/24 dev veth-red
sudo ip netns exec green ip addr add 192.168.1.2/24 dev veth-green
```

**Step 8:** Activate the IP links for each namespace.

```bash
sudo ip netns exec red ip link set veth-red up
sudo ip netns exec green ip link set veth-green up
```

We can accetive the loopback as well

```bash
sudo ip netns exec red ip link set lo up
sudo ip netns exec green ip link set lo up
```

**Step 9:** Now its time to Up the bridge connection as well,

```bash
sudo ip link set dev veth-red-br up
sudo ip netns set dev veth-green-br up
```

**Step 10:** Now its time to test the connection again

Execute the following command to ping `green` from `red`:

```bash
sudo ip netns exec red ping 192.168.1.2
```

Execute the following command to ping `red` from `green`:

```bash
sudo ip netns exec green ping 192.168.1.1
```

After connecting a device to the network, we can see some connection information by pinging and checking the ARP table. Let's explore this by entering a network namespace and interacting with the default network device.

Most systems have `eth0` as the main device, but in my case it is `enp0s1`. To view the device IP, we can use:

```bash
sudo ip addr
```

My IP is `192.168.64.3/24`. Now let's try pinging this from a namespace called red:

```bash
sudo ip netns exec red bash
ping 192.168.64.3
```

You'll likely see a "network unreachable" message because there is no default gateway set for the namespace. We can check the routes with:

```bash
route
```

Now let's set the default gateway to point to our bridge network:

```bash
ip route add default via 192.168.1.15/24
```

With that set, we can now ping the root device:

```bash
ping 192.168.64.3
```

However, pinging an external IP like Google will still fail:

```bash
ping 8.8.8.8
```

This happens because our private network isn't known to the public IP space. We need firewall rules to do Source NAT (SNAT) in the nat table to masquerade our IP range:

```bash
sudo iptbales -t nat -A POSTROUTING -s 192.168.1.0/24 -j MASQUERADE
```

Now traffic from our private range will be SNAT'd when routed externally.

Hence we can now ping to the google

```bash
ping 8.8.8.8
```
