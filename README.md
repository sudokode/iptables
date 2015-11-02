Issues and patches are welcome. Released under Creative Commons ([CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)).

This aims to provide an introduction to iptables for moderately advanced (or above) users that have never seen iptables before. This is a cursory overview of iptables with some basic examples.

Legend:
  * **commonly used or referred to**
  * ~~not commonly used or not documented here~~

[iptables](http://netfilter.org/projects/iptables/index.html)
==========

`iptables` (and `ip6tables`) is a userspace tool used to configure NAT, packet mangling, and stateful packet filtering within the Linux kernel's Xtables/Netfilter framework. It uses organizational units called tables to hold chains that hold rules, all of which make up a ruleset. Before writing any rules, it's important to set a few definitions.

[Statefulness](https://github.com/sudokode/iptables/blob/master/states.rules)
--------------
Netfilter is a stateful packet filter framework, which means it organizes packets into groups called connections described by attributes like source and destination addresses. Be advised: these connections only represent logical organizational packets used by Netfilter and not the same connection used by protocols like TCP or SCTP. Within Netfilter, even ICMP and UDP packets are part of a connection, and it is important to understand when various states of connection tracking occur, independent of the underlying protocol. The `conntrack` tool is useful for examining connections.

There are five main states and two virtual states that connections within Netfilter will fall:
  * **NEW** - traffic only seen in one direction, in or out (e.g. TCP SYN)
  * **ESTABLISHED** - traffic seen in both directions, in and out (e.g. TCP ACK)
  * **RELATED** - new connection from an ESTABLISHED connection (e.g. FTP transfer)
  * INVALID - no connection (e.g.  TCP ACK without a prior SYN)
  * UNTRACKED - not tracked
  * ~~SNAT~~ - original source address differs from the reply destination
  * ~~DNAT~~ - original destination address differs from the reply source

[Tables](https://github.com/sudokode/iptables/tree/master/tables) (-t, -L, -S, -F, -Z, -X)
--------
Tables contain chains. There are four tables, only two of which are used during normal configuration:
  * [**filter**](https://github.com/sudokode/iptables/blob/master/tables/filter.rules) - where the firewall stuff goes: accept, drop, reject, rate limit, etc
  * [**nat**](https://github.com/sudokode/iptables/blob/master/tables/nat.rules) - where the NAT stuff goes: SNAT, DNAT, masquerade, redirection, etc
  * [mangle](https://github.com/sudokode/iptables/blob/master/tables/mangle.rules) - where you mangle packets: change TTL, etc
  * ~~raw~~ - where you shouldn't be

[Chains](https://github.com/sudokode/iptables/tree/master/chains) (-A, -C, -D, -I, -R, -L, -S, -F, -Z, -X, -P, -E)
--------
Chains contain rules. There are five built-in chains that exist on various tables:
  * [**INPUT**](https://github.com/sudokode/iptables/blob/master/chains/INPUT.rules) - filter, mangle
  * [OUTPUT](https://github.com/sudokode/iptables/blob/master/chains/OUTPUT.rules) - filter, nat, mangle, raw
  * [**FORWARD**](https://github.com/sudokode/iptables/blob/master/chains/FORWARD.rules) - filter, mangle
  * [PREROUTING](https://github.com/sudokode/iptables/blob/master/chains/PREROUTING.rules) - nat, mangle, raw
  * [**POSTROUTING**](https://github.com/sudokode/iptables/blob/master/chains/POSTROUTING.rules) - nat, mangle

Custom chains can be added for more organization.

Traversal
---------
It's far easier to understand if you just look at [this image](https://www.frozentux.net/iptables-tutorial/images/tables_traverse.jpg). Focus on the **filter** and **nat** tables as they will contain most of your rules, but understand that every packet travels through every one of those tables/chains in that order every time (except where certain things like NAT come into play). Each packet also has to traverse each rule in each chain, so it's a good idea to put the most important or commonly used rules at the top and not to go overkill on custom chains that take up time jumping back and forth (more on that in the next section). *Protip: The easiest way to speed up your ruleset and save some time typing is to isolate the NEW state by accepting or dropping the others, leaving you with the very normal firewall task of whitelisting ports you want to listen for new connections on. (More on this later.)*

Rules
-----
Think of a rule as a sentence containing a subject and a predicate. But instead, a rule contains a match (or matches) and a target (jump). All of the conditions must be met in order for the jump to take place. A jump will always lead to another chain. In the case of built-in jumps, you will end up at the next chain in the normal traversal path. A custom chain behaves similarly to a function in a script. A jump is used (like a function call) to move to that chain and after its traversal, the flow jumps back to the next rule in the calling chain. The same applies for all rules in all chains on all tables.

[Matches](https://github.com/sudokode/iptables/tree/master/matches)
---------
Matches are used to narrow down rules by describing attributes of a packet, such as its source address or connection state. Matches come in three flavors:
  * generic - generic protocol matches
  * implicit - protocol-specific matches
  * explicit - matches not related to protocols

[**Generic matches**](https://github.com/sudokode/iptables/blob/master/matches/generic.rules):
  * **-p: protocol (tcp, udp, icmp, sctp)**
  * **-s: source address(es)**
  * **-d: destination address(es)**
  * -i: input interface
  * -o: output interface
  * ~~-f: packet fragment~~

[**Implicit matches**](https://github.com/sudokode/iptables/blob/master/matches/implicit.rules):
  * **--sport: source port (TCP, UDP, SCTP)**
  * **--dport: destination port (TCP, UDP, SCTP)**
  * --tcp-flags: TCP flags
  * --syn: TCP SYN packet
  * ~~--tcp-option~~: TCP option
  * **--icmp-type: ICMP type**
  * ~~--chunk-types~~: SCTP chunk type
  
[**Explicit matches**](https://github.com/sudokode/iptables/blob/master/matches/explicit.rules) (-m <match>), abridged list:
  * comment - add comments to the ruleset
  * connmark - matches a marked connection (-j CONNMARK)
  * **conntrack** - matches connection tracking attributes like state
  * **iprange** - matches a range of IP addresses
  * ~~length~~ - matches packet length
  * ~~limit~~ - rate limiting
  * ~~mac~~ - matches MAC address
  * **multiport** - matches a list of ports
  * ~~owner~~ - matches process attributes
  * **recent** - matching system, commonly used for rate limiting
  * ~~ttl~~ - matches packet TTL

Jumps (-j)
-----
Jumps are actions to take once a match is achieved. Some matches, like comment, do not require a jump; the module does the work already. Those are rare, so at the end of most rules, be prepared to jump. A jump can be a custom chain or a target. You cannot jump to the built-in chains so as not to interfere with normal traversal.

Targets (-j)
-------
Chains can be jumped to within tables, but to do certain actions once a final match is achieved, targets are used. Here are some commonly used targets:
  * **ACCEPT** - allows a packet to continue
  * CONNMARK - marks a connection
  * DNAT - performs destination NAT on a packet
  * **DROP** - immediately drops a packet
  * **LOG** - logs a packet
  * **MASQUERADE** - SNAT without a specified source address
  * NOTRACK - disables connection tracking
  * REJECT - drops a packet gracefully
  * **SNAT** - performs source NAT on a packet
  * TTL - change packet's TTL

Default Policies (-P)
----------------
The default policy of a chain is a target that the chain ends with by default. And by default, every chain is set to ACCEPT. The only other valid value is DROP; the REJECT target cannot be used. Custom chains cannot have default policies as they will only ever be called from the built-in chains anyway.

iptables
--------
Yes, the tool again. Most of the topics from above are simply flags you pass to `iptables`:
  * -t: select table (defaults to filter)
  * -A: append rule to chain
  * -D: delete rule from chain
  * -I: insert rule into chain
  * -L: list rules [in chain]
  * -S: list rules \[in chain\] (rules file format)
  * -j: jump to target/chain
  * -P: set default policy

All of these flags are used with `iptables` to ever so slowly configure your firewall...

Ruleset (Rules File)
-------
Once you understand the basics, you can put your entire ruleset into a file rather than using `iptables`. The format is simple:

    # iptables ruleset - 2015/11/02
    *nat
    rule1
    rule2
    COMMIT

    *filter
    rule1
    rule2
    COMMIT

Each rule is simply the equivalent `iptables` command without the `iptables` part. So to demonstrate for real this time:

    # iptables ruleset - 2015/11/02
    *nat
    -A POSTROUTING -o eth0 -j MASQUERADE
    COMMIT

    *filter
    -P INPUT DROP
    -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -s 127.0.0.1 -j ACCEPT
    COMMIT

If this looks scary, **pay attention** because here it is line by line:
  1. This is a comment which will not end up in the ruleset presented to the kernel (there's a way to do that, look up).
  2. In a rules file, this sets the table you're in (instead of using -t).
  3. This line allows for masquerading on packets leaving out of the eth0 interface. This is a more advanced configuration, something a router might do.
  4. Similar to line 2, this is necessary to close a table in a rules file.
  5. See line 2.
  6. This sets the default policy of the INPUT chain to DROP.
  7. This matches any packets in the ESTABLISHED and RELATED state and accepts them. Remember, isolate the NEW state.
  8. This matches any packet with a source address of 127.0.0.1 (our loopback address) and accepts it. Don't forget to do this.
  9. See line 4.

That is a fairly basic ruleset, but it's actually not far off from the most basic firewall setup:

    # iptables ruleset - 2015/11/02
    *filter
    -P INPUT DROP
    -P FORWARD DROP
    -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -i lo -j ACCEPT
    COMMIT

Very easy to see what's changed and how. Set the default policies, isolate the NEW state, accept loopback, done. That's literally all there is to it. From there, you simply build bigger rules.

iptables-restore
----------------
`iptables-restore` is the handy tool that takes your rules file, parses it, and loads it atomically into the kernel. Once you have the rules from above in a file, you can use this:
  `iptables-restore /path/to/ruleset`

Further Reading
---------------
* [Netfilter iptables homepage](http://netfilter.org/projects/iptables/index.html)
* [Great iptables tutorial from FrozenTux](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
* [iptables(8)](http://ipset.netfilter.org/iptables.man.html)
* [iptables-extensions(8)](http://ipset.netfilter.org/iptables-extensions.man.html)
* [ArchWiki](https://wiki.archlinux.org/index.php/Iptables)
* [Wikipedia](https://en.wikipedia.org/wiki/Iptables)

