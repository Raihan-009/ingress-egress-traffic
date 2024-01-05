# linux-ns-networking-4


## `Ping 8.8.8.8 from ns0` [ egress traffic ]

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

## `Role of POSTROUTING`

The POSTROUTING chain in iptables is part of the NAT (Network Address Translation) table, and it is applied to packets after the routing decision has been made but before the packets are sent out of the system. The primary purpose of the POSTROUTING chain is to perform Source Network Address Translation (SNAT) or MASQUERADE on outgoing packets.

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

These rules enabled traffic to travel across the br0 virtual bridge.These are useful to allow all traffic to pass through the br0 interface without any restrictions. However, keep in mind that using such rules without any filtering can expose your system to potential security risks. But for now we re good to ping!

## `Test Connectivity`

```bash
ping 8.8.8.8 -c 5
```

![ping-pong](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/6b93a385-2c66-4c92-a5c4-df8745dad72d.png)

Bingo! 


## `Ping ns0 from outside world` [ ingress traffic ]

Let's run a server inside the `ns0` namespace.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World! :)'

if __name__ == '__main__':
    app.run(host='192.168.0.2', port=3000, debug=True)
```

## `Check Accessibility`

Let's try to access from RootNS using `telnet` linux utility by establishing a new SSH session.

```bash
telnet 192.168.0.2 3000
```

Also, we can use curl for making HTTP requests.

```bash
curl 192.168.0.2:3000
```

![process](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/9e479d5f-a810-48d4-9d52-3b103367ccd3.png)

Cheers! We have received a response from our server, and everything is up and running. Now let's try from outside the world using the `public IP` of my VM. My `public ip` is 36.255.70.217.

```bash
telnet 36.255.70.217 3000
```
The output we have got. 

```bash
Trying 36.255.70.217.
telnet: connect to address 36.255.70.217: Connection refused
telnet: Unable to connect to remote host
```
Now let's try from the browser too.

![browser](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/a02a5236-02c6-45bd-97d3-113290895bd3.png)

Why are we not getting a response from our server, or why has the server refused to connect?

## `Root Cause`

Here, when we tried to reach our process, which is located inside the custom namespace, our VM didn't know where to forward the traffic.

## `Solution`

In order to resolve this, we need to add the DNAT (Destination Network Address Translation) rule. In short, we need to specify where the packets need to be redirected when they are trying to reach a specific port with our public IP.

```bash
sudo iptables -t nat -A PREROUTING -d 10.0.0.25 -p tcp -m tcp --dport 3000 -j DNAT --to-destination 192.168.0.2:3000
```

Let's verify the iptables rule
```bash
sudo iptables -t nat -L -n -v
```

We will see something like

```bash
Chain PREROUTING (policy ACCEPT 10 packets, 508 bytes)
 pkts bytes target     prot opt in     out     source               destination
 230K   12M DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            10.0.0.25            tcp dpt:3000 to:192.168.0.2:3000
```

## `Role of PREROUTING`

The `PREROUTING` chain is executed before the Linux kernel makes the routing decision for an incoming packet. It allows us to alter the packet's destination address before the packet is routed to its final destination. The rule modifies the destination address and port of incoming packets with a destination IP of `10.0.0.25` and a destination port of `5000`. It changes the destination to `192.168.0.2:5000`.

## `Role of DNAT`

Here `DNAT` is used to change the destination address of a packet. When a packet matches the specified conditions (destination IP is `10.0.0.25` and destination port is `5000`), DNAT is triggered, and the packet's destination is changed to `192.168.0.2:5000`.

So now, we are good to ping!

## `Test Connectivity`

Let's start with linux utility `telnet` and `curl`.

![curl](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/f3958f8b-fad3-4ff9-a755-de1db21cbd20.png)

One last time from our browser.

![browser](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/cb642c85-9a5b-4629-9e31-56d3bbb8077d.png)

Yaahoo!! We have successfully established both ingress and egress traffic with linux network namespace. :)