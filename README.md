# linux-ns-networking-4


## `Ping 8.8.8.8 from ns0`

We have come so far. Now we can ping our root ns or primary ethernet interface from custom namesapce. But can we ping outside the world? Let's ping to 8.8.8.8.

```bash
sudo ip netns exec ns0 bash
ping 8.8.8.8 -c 5
```

Okay, it does not seem that the network is unreachable. It's something else. Maybe somewhere, packets have stuck. Let's dig it out. We can do it using the Linux utility `tcpdump`. We will check two interfaces. One is `br0`, and the other is `ens3` (in the case of my device).
 

![stuck](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/07e5412b-3ec3-4e43-8893-e0c2ed7fef9c.png)

Here we can see that packets are coming to those interfaces but are still stuck in `ns0`. But why?

## `Root Cause`

See! The source IP address, 192.168.0.2, is attempting to connect to Google DNS 8.8.8.8 using its `private IP address`. So it's very basic that the outside world can't reach that private IP because they have no idea about that particular private IP address. That's why packets are able to reach Google DNS, but we are not getting any replies from 8.8.8.8.

## `Solution`

We must somehow change the private IP address to a public address in order to sort out this issue with the help of NAT (Network Translation Address). So, a SNAT (source NAT) rule must be added to the IP table in order to modify the POSTROUTING chain.


```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/16 ! -o br0 -j MASQUERADE
```

## `MASQUERADE Action`

The MASQUERADE target in iptables is used for Network Address Translation (NAT) in order to hide the true source address of outgoing packets. Here, when the packet matches the conditions specified in an iptables rule with the MASQUERADE target, the source IP address of the packet is dynamically replaced with the IP address of the outgoing interface. The NAT engine on the router or gateway replaces the private source IP address with its own public IP address before forwarding the packet to the external network. When the external network sends a reply back to the public IP address, the NAT engine tracks the translation and forwards the reply back to the original private IP address within the internal network.

## `Firewall rules`

```bash
sudo iptables -t nat -L -n -v
```

![iptable](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/632ad080-cb31-4dee-8b03-1141fb1707b6.png)

In case if still it not works then we may need to add some additional firewall rules.

```bash
sudo iptables --append FORWARD --in-interface br0 --jump ACCEPT
sudo iptables --append FORWARD --out-interface br0 --jump ACCEPT
```

These rules enabled traffic to travel across the v-net virtual bridge.These are useful to allow all traffic to pass through the v-net interface without any restrictions.However, keep in mind that using such rules without any filtering can expose your system to potential security risks. But for now we re good to ping!

## `Test Connectivity`

```bash
ping 8.8.8.8 -c 5
```

![ping-pong](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/6b93a385-2c66-4c92-a5c4-df8745dad72d.png)

Bingo! 
