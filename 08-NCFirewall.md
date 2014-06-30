Chapter 8: Firewall Redux
=========================

In [Chapter 3](03-OxFirewall), you wrote a firewall that blocks ICMP traffic using OpenFlow and Ox. You did this in two steps: first, you wrote a _packet_in_ function and then configured the flow table to implement the same function efficiently. 
This single-line Frenetic program performs the same function: 

`if nwProto = 1 then drop else all`. 

In this chapter, you'll implement a more interesting firewall policy. This time, you will still use a trivial, one-switch topology. But, in the next chapter, you'll see 
how to turn your firewall into an easy-to-reuse component that can be applied to any other topology.

## Topology 

You are going to program the following network of four hosts and one switch:

![image](images/topo-single-4.png)

The host with MAC address `00:00:00:00:00:0n` is connected to port `n`. Mininet has builtin support for building single-switch topologies:

```
$ sudo mn --controller=remote --topo=single,4 --mac --arp
```

### Exercise 1: Forwarding

Write a forwarding policy for this network. Use `monitorTable` to examine the flow table that the compiler generates. Try a few `ping`s between hosts.

As you've seen, Frenetic supports ordinary `if`-`then`-`else` expressions.
So, you can implement the forwarding policy as follows:

```
let forwarding =
  if dlDst=00:00:00:00:00:01 then
     fwd(1)
  else if (* destination is 2, forward out port 2, etc. *)
    ...
  else
    drop
    
monitorTable(1, forwarding)
```

Fill in the rest of the policy by editing `frenetic-tutorial-code/Chapter7.nc`.

#### Testing

Launch Frenetic in one terminal:

```
$ frenetic Chapter7.nc
```

And Mininet in another:

```
$ sudo mn --controller=remote --topo=single,4 --mac --arp
```

Using Mininet, ensure that you can ping between all hosts:

```
mininet> pingall
```

## Firewall Policy

Now that basic connectivity works, you should enforce the access control (firewall) policy written in the table below:

<table>
<tr>
  <th style="visibility: hidden"></th>
  <th style="visibility: hidden"></th>
  <th colspan="4">Dst MAC address</th>
</tr>
<tr>
  <th style="visibility: hidden"></th>
  <th style="visibility: hidden"></th>
  <th>00:00:00:00:00:01</th>
  <th>00:00:00:00:00:02</th>
  <th>00:00:00:00:00:03</th>
  <th>00:00:00:00:00:04</th>
</tr>
<tr>
  <th rowspan="5" style="-webkit-transform:rotate(270deg)" >
    Src MAC<br>address
  </th>
  <th>00:00:00:00:00:01</th>
  <td>Deny All</td>
  <td>HTTP</td>
  <td>Deny All</td>
  <td>Deny All</td>
</tr>
<tr>
  <th>00:00:00:00:00:02</th>
  <td>HTTP</td>
  <td>Deny All</td>
  <td>Deny All</td>
  <td>Deny All</td>
</tr>
<tr>
  <th>00:00:00:00:00:03</th>
  <td>Deny All</td>
  <td>Deny All</td>
  <td>Deny All</td>
  <td>ICMP</td>
</tr>
<tr>
  <th>00:00:00:00:00:04</th>
  <td>Deny All</td>
  <td>Deny All</td>
  <td>ICMP</td>
  <td>Deny All</td><br>
</tr>
</table>

Each cell in this table has a list of allowed protocols for connections between
clients (rows) and servers (columns). For example, consider this entry in the table:


<table>
<tr>
  <th></th>
  <th>00:00:00:00:00:01</th>
</tr>
<tr>
  <th>00:00:00:00:00:02</th>
  <td>HTTP</td>
</tr>
</table>

This cell indicates that (only) HTTP connections (port 80) are allowed between client
`00:00:00:00:00:02` and the server `00:00:00:00:00:01`. To realize this policy in Frenetic, you need to allow packets from the client to port 80 on the server *and* from port 80 on the server to the client:

```
if (dlSrc = 00:00:00:00:00:02 && dlDst = 00:00:00:00:00:01 && tcpDstPort = 80) ||
   (dlSrc = 00:00:00:00:00:01 && dlDst = 00:00:00:00:00:02 && tcpSrcPort = 80)
then
  forwarding
else
  drop
```


### Exercise 2: Firewall + Forwarding

Wrap the forwarding policy you wrote above within a policy implementing the firewall.
Assume standard numberings:

- HTTP packets are on port 80 and 
- ICMP packets are nwProto 1.

> See `frenetic-tutorial-code/Sol_Chapter7_Forwarding.nc`, if you
> did not finish the previous task.

Your edited file will probably have the following structure:

```
let forwarding = (* from part 1, above *)

let firewall =
  if (* traffic is allowed *) then
    forwarding
  ...
  else
    drop

firewall
```

While you could write the policy by enumerating each allowed flow, consider
using `if`-`then`-`else` and boolean expressions (`p1 && p2`, `p1 || p2`, and `!p`) to write a compact and legible policy.

#### Testing

Launch Frenetic in one terminal:

```
$ frenetic Chapter7.nc
```

And Mininet in another, then open a terminal on each host:

```
$ sudo mn --controller=remote --topo=single,4 --mac --arp
mininet> xterm h1 h2 h3 h4
```

Instead of trying a comprehensive test, just test a few points of the access control policy. For example, if you run _fortune_ on port 80 on `h1`:

```
## Run on h1's terminal
$ while true; do fortune | nc -l 80; done
```

Then, running `curl 10.0.0.1:80` should succeed from `h2`, but fail from `h3`.

Similarly, pinging `h3` should succeed from `h4`, but fail from `h1`.

## Next chapter: [Multi-switch Programming][Ch8]

[Ch8]: 08-NCMultiSwitch

[Action]: http://frenetic-lang.github.io/frenetic/docs/OpenFlow0x01.Action.html