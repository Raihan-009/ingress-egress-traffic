# linux-ns-networking-4

We have come so far. Now we can ping our root ns or primary ethernet interface from custom namesapce. But can we ping outside the world? Let's ping to 8.8.8.8.

```bash
sudo ip netns exec ns0 bash
ping 8.8.8.8 -c 5
```
 
Okay, it does not seem that the network is unreachable. It's something else. Maybe somewhere, packets have stuck. Let's dig it out. We can do it using the Linux utility `tcpdump`. We will check two interfaces. One is `br0`, and the other is `ens3` (in the case of my device).
 

![stuck](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/07e5412b-3ec3-4e43-8893-e0c2ed7fef9c.png)

Here we can see that packets are coming to those interfaces but are still stuck in `ns0`. But why?