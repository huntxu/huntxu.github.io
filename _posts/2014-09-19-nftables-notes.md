---
layout: post
title: nftables 学习笔记
---

## Introduction

### WHAT

1. kernel 3.13
2. nft, syntax different to iptables
3. compatibility layer for iptables
4. set infrastructure for mappings between matchings and actions
5. consider WIP


### difference with iptables
1. syntax
  * iptables - getopt_long()-based
  * nftables - tcpdump-inspired
2. configurable tables and chains
3. matches and targets no distinction
4. multiple actions in one rule
5. no built-in counter
6. generic set infrastructure
7. new protocols without kernel upgrades -- pseudo-state machine


### netfilter hooks
                                                  Local machine
                                                   ^         |                 .-----------.
                        .-----------.              |         |                 |  Routing  |
                        |           |-----> Input /           \---> Output --->|  Decision |------
    ---> Prerouting --->|  Routing  |                                          |-----------|      \
                        | Decision  |                                                              ---> Postrouting --->
                        |           |                                                             /
                        |           |---------------> Forward ------------------------------------
                        |-----------|


## Basic

### tables
1. no predefined table
2. kind of tables depending on the family
  * ip
  * arp
  * ip6
  * bridge
  * inet (Linux 3.14) -- IPv4 + IPv6
3. adding
  * nft add table FAMILY NAME -- nft add table ip filter
4. listing
  * nft list tables FAMILY -- nft list tables ip
  * nft list table FAMILY NAME -- nft list table ip filter
5. deleting
  * nft delete table FAMILY NAME -- nft delete table ip filter (only works if the table does not contain any chain)
6. flushing
  * nft flush table FAMILY NAME -- nft flush table ip filter


### chains
1. base chains -- registered into the netfilter hooks
2. adding
  * nft add chain ip filter input { type filter hook input priority 0 \; }
  * registers the input chain, attached to the input hook
  * priority
  * nft add chain ip filter output { type filter hook output priority 0 \; }
3. types
  * filter - filter packets, arp, bridge, ip, ip6, inet
  * route - reroute if relevant IP head field or the packet mark is modified, ip, ip6
  * nat - only the first packet of a flow hits this chain, not for filtering, ip, ip6
4. hooks
  * prerouting
  * input
  * forward
  * output
  * postrouting
5. priority - used to order the chains or to put them before or after some netfilter internal operations
  * NF_IP_PRI_CONNTRACK_DEFRAG (-400): priority of defragmentation 
  * NF_IP_PRI_RAW (-300): traditional priority of the raw table placed before connection tracking operation 
  * NF_IP_PRI_SELINUX_FIRST (-225): SELinux operations 
  * NF_IP_PRI_CONNTRACK (-200): Connection tracking operations 
  * NF_IP_PRI_MANGLE (-150): mangle operation 
  * NF_IP_PRI_NAT_DST (-100): destination NAT 
  * NF_IP_PRI_FILTER (0): filtering operation, the filter table 
  * NF_IP_PRI_SECURITY (50): Place of security table where secmark can be set for example 
  * NF_IP_PRI_NAT_SRC (100): source NAT 
  * NF_IP_PRI_SELINUX_LAST (225): SELInux at packet exit 
  * NF_IP_PRI_CONNTRACK_HELPER (300): connection tracking at exit
6. non-base chains
  * nft add chain ip filter NAME
7. deleting
  * nft delete chain ip filter input
8. flushing
  * nft flush chain ip filter input
9. example
  * nft add table ip filter
  * nft add chain ip filter input { type filter hook input priority 0 \; }
  * nft add chain ip filter output { type filter hook output priority 0 \; }


### rules
1. adding

        nft add rule   filter      output     ip daddr 8.8.8.8    counter
                      ^-table-^   ^-chain-^   ^^--matching--^^   ^-action-^

2. listing
  * nft list table filter
  * nft add rule filter output tcp dport ssh counter
  * disable host name resolution by -n
  * disable service name resolution by -nn
3. testing?
4. rule position
  * rule handle: -a
  * nft add rule filter output position 8 ip daddr 127.0.0.8 drop # adding after rule with handle 8
  * nft insert rule filter output position 8 ip daddr 127.0.0.8 drop # adding before rule with handle 8
5. deleting
  * nft delete rule filter output handle 8 # delete rule with handle 8
  * NOT IMPLEMENTED YET: nft delete rule filter output ip saddr 192.168.1.1 counter # delete such a rule
6. delete all rules
  * in a chain: nft delete rule filter output
  * in a table: nft flush table filter
7. prepending
  * nft insert rule filter output ip daddr 127.0.0.1 counter
8. atomic rule replacement
  * nft -f file
  * nft list table filter > filter-table
  * nft -f filter-table
  * ATOMIC in one single transaction(kernel), shell scripts CANNOT achieve this


### building rules through expressions
  * ne(!=), lt(<), gt(>), le(<=), ge(>=)
  * < and > should be escape in shell
  * example:
    * nft add rule input test test tcp dport != 22
    * nft add rule input test test tcp dport >= 1024


### exporting/importing rules
  * native
    * exporting: nft list table mytable > table.nft
    * importing: nft -f table.nft
  * XML/JSON
    * exporting(all tables of all families):
      * nft export xml > ruleset.xml
      * nft export json > ruleset.json
    * importing: WIP


## Matching

### packet header
  * layer 4 protocols: AH, ESP, UDP, UDPlite, TCP, DCCP, SCTP, IPComp
  * transport protocol(protocol): nft add rule filter output ip protocol tcp
  * ethernet header(ether): nft add rule filter input ether daddr ff:ff:ff:ff:ff:ff counter (eth info only available in the input path)
  * ipv4 header(ip): nft add rule filter input ip saddr 192.168.1.100 ip daddr 192.168.1.1 counter (input chain and daddr?)
  * ipv6 header(ip6): nft add rule filter output ip6 daddr abcd::100 counter (ip6 table and corresponding chains)
  * TCP/UDP/UDPlite traffic:
    * nft add rule filter input tcp dport 1-1024 counter drop
    * nft add rule filter input tcp flags != syn counter
    * nft -i
      nft> add rule filter output tcp flags & (syn | ack) == (syn | ack) counter log
  * icmp(icmp): nft add rule filter input icmp type echo-request counter drop

### packet metainformation
  * selectors
    * interface device name/index: iifname, oifname, iif, oif (index might be changed but the performance is better)
    * interface type: iiftype, oiftype
    * tc handle: priority
    * socket uid/gid: skuid, skgid
    * packet length: length
  * by interface name (index faster(32-bits unsigned integer vs str) but dynamically allocated)
    * nft add rule filter input meta oifname lo accept
  * by packet mark
    * nft add rule filter output meta mark 123 counter
  * by socket uid/gid
    * nft add rule filter output meta skuid hunt counter (username)
    * nft add rule filter output meta skuid 1000 counter (UID)
    * example:
      * wget --spider http://www.google.com
      * nft list table filter
      * ping example?

### conntrack metainformation
  * selector: ct
  * states: new, established, related, invalid
    * nft add rule filter input ct state established,related counter accept
    * nft add rule filter input counter drop
  * the conntrack mark
    * nft add rule filter input ct mark 123 counter (how to set the mark (or any other packet metainformation))

### rate limiting matchings
  * nft add rule filter input icmp type echo-request limit rate 10/second accept


## actions
* several actions in one single rule

### accepting and dropping
  * dropping
    * terminating action
    * nft add rule filter output drop
  * accepting
    * nft add rule filter output accept
    * adding counters: nft add rule filter output counter accept

### jumping
  * nft add chain ip filter tcp-chain
  * nft add rule ip filter input ip protocol tcp jump tcp-chain
  * nft add rule ip filter tcp-chain counter
  * nft list table filter

### rejecting
  * nft add rule filter input reject
  * nft delete rule filter input
  * nft add rule filter input ct state new reject

### logging
  * nft add rule filter input tcp dport 22 ct state new log prefix \"SSH for ever: \" accept

### nat
  * special semantics
    * first packet of a flow: used to look up a matching rule which sets up the NAT binding for the flow (and also manupulates the first packet).
    * follow up packets in the flow: no rule lookup, the NAT engine uses the NAT binding information to perform packet manipulation
  * source NAT
    * match traffic from 192.168.1.0/24 to eth0, use 1.2.3.4 as source for the packets
      * nft add table nat
      * nft add chain nat prerouting { type nat hook prerouting priority 0 \; }
      * nft add chain nat postrouting { type nat hook postrouting priority 0 \; }
      * nft add rule nat postrouting ip saddr 192.168.1.0/24 oif eth0 snat 1.2.3.4
  * destination NAT
    * redirect incoming traffic via eth0 for TCP ports 80 and 443 to 192.168.1.120
      * nft add table nat
      * nft add chain nat prerouting { type nat hook prerouting priority 0 \; }
      * nft add chain nat postrouting { type nat hook postrouting priority 0 \; }
      * nft add rule nat prerouting iif eth0 tcp dport { 80, 443 } dnat 192.168.1.120
  * masquerading
    * Linux 3.18
  * incompatibilities
    * cannot use iptables and nft NAT at the same time
    * modprobe -r iptable_nat

### setting packet metainformation
  * Linux 3.14+
  * mark
    * nft add rule route output mark set 123
  * mark and conntrack mark
    * nft add rule filter forward meta mark 1
    * nft add rule filter forward ct mark set mark
    * nft add rule filter forward meta mark set ct mark
  * priority
    * nft add table mangle
    * nft add chain postrouting { type route hook output priority -150\; }
    * nft add rule mangle postrouting tcp sport 80 meta priority set 1
  * nftrace
    * nft add rule filter forward udp dport 53 meta nftrace set 1
  * combination
    * nft add rule filter forward ip saddr 192.168.1.1 meta nftrace set 1 meta priority set 2 meta mark set 123

### queueing
  * libnetfilter_queue
    * nfqnl_test
      * libnetfilter_queue/utils
      * nfqnl_test QUEUENUM
  * command
    * nft add filter input counter queue
    * nft add filter input counter queue num 3
  * no userspace application listening -> drop
  * bypass: no userspace application listening -> accept
    * nft add filter input counter queue bypass
  * load balance traffic to several queues
    * nft add filter input counter queue num 0-3
    * fanout: use the CPU ID as an index to map packets to the queues
      * improve performance if queue/userspace application per CPU
      * nft add filter input counter queue num 0-3 fanout
  * combination
    * nft add filter input counter queue num 0-3 fanout bypass

### counter 
  * optional in nftables
  * position in the rule syntax
    * nft add rule filter input ip protocol tcp counter
    * nft add rule filter input counter ip protocol tcp (left-to-right evaluation)


## advanced data structures

### sets
  * overview
    * use ANY supported selector to build sets
    * make dictionaries and maps possible
    * internal: use hashtables and red-black trees
  * anonymous sets
    * bound to a rule: release when the rule is removed
    * no specific name: kernel-allocated identifier
    * CANNOT be updated: cannot add or delete elements
    * example
      * nft add rule filter output tcp dport { 22, 23 } counter
  * named sets
    * example
      * nft add set filter blackhole { type ipv4_addr\; }
      * nft add element filter blackhole { 192.168.3.4 }
      * nft add element filter blackhole { 192.168.1.4, 192.168.1.5 }
      * nft add rule ip input ip saddr @blackhole drop
    * supported data types
      * IPv4 address
      * IPv6 address
      * Ethernet address
      * Protocol type
      * Mark type
    * CAN be updated: add and delete elements
    * listing content of a named set
      * nft list set filter MYSET
  * WHY nftables
    * iptables
      * ip6tables -A INPUT -p tcp -m multiport --dports 22,80,443 -j ACCEPT
      * ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation -j ACCEPT
      * ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -j ACCEPT
      * ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
      * ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement -j ACCEPT
    * nftables
      * nft add rule ip6 filter input tcp dport {telnet, http, https} accept
      * nft add rule ip6 filter input icmpv6 type { nd-neighbor-solicit, echo-request, nd-router-advert, nd-neighbor-advert } accept

### dictionaries
  * aka verdict maps
  * example
    * nft add rule ip filter input ip protocol vmap { tcp : jump tcp-chain, udp : jump udp-chain , icmp : jump icmp-chain } (the chains are assumed to be created)
    * nft add rule ip filter input counter drop
    * nft add rule filter icmp-chain counter
    * nft add rule filter tcp-chain counter
    * nft add rule filter udp-chain counter
    * nft list table filter

### intervals
  * example
    * nft add rule filter input ip daddr 192.168.0.1-192.168.0.250 drop
    * nft add rule filter input tcp ports 1-1024 drop
  * use in sets
    * nft add rule ip filter input ip saddr { 192.168.1.1-192.168.1.200, 192.168.2.1-192.168.2.200 } drop
  * use in dictionaries
    * nft add rule ip filter forward ip daddr { 192.168.1.1-192.168.1.200 : jump chain-dmz, 192.168.2.1-192.168.20.250 : jump chain-desktop } drop

### maps
  * example
    * nft add rule ip filter input dnat set tcp dport map { 80 : 192.168.1.100, 8888 : 192.168.1.101 }


- - -
* https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE ??
