# Modern-Network-Cycle

## Laptop Joins LAN

Laptop authenticates to Wi-Fi or plugs into Ethernet.

Now the interface is UP, but no IPs yet.

## IPv6: Create Link-Local Address (LLA)

1. Laptop generates IPv6 LLA

Tentative LLA: fe80::abcd:ef12:3456:7890

No packets yet — purely local process.

State now: Laptop has tentative IPv6 LLA, not yet usable.

## IPv6 Duplicate Address Detection (DAD) for LLA

2. {ICMPv6 NS — DAD for LLA}

Protocol: ICMPv6 Neighbor Solicitation (Type 135)

Purpose: Check if LLA is unique

Src IP: :: (unspecified)

Dst IP: ff02::1:ffXX:XXXX (solicited-node multicast of tentative LLA)

Src MAC: Laptop MAC

Dst MAC: 33:33:ff:XX:XX:XX (IPv6 multicast MAC)

Laptop does NOT include SLLA option (as required for DAD).

3. Response: None expected

If no ICMPv6 NA is received within timeout → address is unique.

📌 Phase summary:

Laptop now has a valid IPv6 LLA:

✔ Can communicate with other IPv6 devices on LAN (LLA only)

✔ Can send RS to the router

## IPv6 Router Discovery (RS → RA)

4. {ICMPv6 RS — Router Solicitation}

Purpose: Request prefixes, DNS, gateway

Protocol: ICMPv6 RS (Type 133)

Src IP: Laptop LLA (fe80::…)

Dst IP: ff02::2 (All-routers multicast)

Src MAC: Laptop MAC

Dst MAC: 33:33:00:00:00:02

5. {ICMPv6 RA — Router Advertisement}

Router replies.

Protocol: ICMPv6 RA (Type 134)

Src IP: Router LLA (fe80::1)

Dst IP: Laptop LLA (unicast) or ff02::1 (multicast)

Includes:

Prefix Information: e.g. 2001:db8:10:20::/64

Router Lifetime: e.g. 1800 sec

MTU Option

Flags:

A = Autonomous (SLAAC)

M = Managed (DHCPv6 address assignment)

O = Other config (DHCPv6 for DNS etc)

RDNSS (IPv6 DNS servers)

Router LLA = Default gateway

📌 Phase summary:

Laptop learned from RA:

✔ IPv6 prefix → can generate GUA

✔ Router = default gateway

✔ IPv6 DNS (RDNSS)

✔ MTU

✔ SLAAC method

## IPv6 GUA Creation + DAD

6. Laptop creates GUA IPv6

Using prefix: e.g. 2001:db8:10:20::/64

Generated GUA: 2001:db8:10:20:abcd:ef12:3456:7890

7. {ICMPv6 NS — DAD for GUA}

Src IP: ::

Dst: ff02::1:ffXX:XXXX of the tentative GUA

Laptop checks uniqueness.

8. Response: None

Address considered unique.

📌 Phase summary:

Laptop now has:

✔ IPv6 LLA → LAN connectivity

✔ IPv6 GUA → WAN (Internet) connectivity

✔ IPv6 DNS (RDNSS)

✔ Default gateway (router LLA)

## IPv4 Address Assignment (DHCPv4)

9. {DHCPDISCOVER}

Protocol: DHCPv4 (UDP 68 → 67)

Src MAC: Laptop MAC

Src IP: 0.0.0.0

Dst IP: 255.255.255.255

Dst MAC: ff:ff:ff:ff:ff:ff

10. {DHCPOFFER}

Protocol: DHCPv4 (UDP 67 → 68)

Src IP: router/DHCP server

Dst IP: 255.255.255.255

Contains:

IPv4 address (192.168.1.X)

Subnet mask

Default gateway

DNS server

Lease time

11. {DHCPREQUEST}

Laptop formally requests offered IP.

Broadcast to 255.255.255.255.

12. {DHCPACK}

Router confirms:

IP lease

DNS

Gateway

13. {ARP Probe / Gratuitous ARP}

Protocol: ARP

Src IP: 0.0.0.0

Src MAC: laptop

Dst MAC: ff:ff:ff:ff:ff:ff

Purpose: check for IP conflict.

📌 Phase summary:

Laptop now has:

✔ IPv4 address

✔ IPv4 mask

✔ IPv4 default gateway

✔ IPv4 DNS

✔ Verified no conflict via ARP probe

## Learn Router MAC Addresses

14. {ICMPv6 NS → Router LLA}

Laptop wants router’s MAC for IPv6 traffic.

Src IP: Laptop GUA

Dst IP: ff02::1:ffXX:XXXX (router’s SNMC)

Dst MAC: 33:33:ff:XX:XX:XX

15. {ICMPv6 NA From Router}

Src IP: fe80::1

Dst IP: laptop GUA

Router includes TLLA option with its MAC.

Laptop stores entry in NDP cache.

16. {ARP Request → Router IPv4}

Src MAC: laptop

Dst MAC: ff:ff:ff:ff:ff:ff

“Who has 192.168.1.1?”

17. {ARP Reply from Router}

Provides router’s MAC.

Laptop stores entry in ARP cache.

📌 Phase summary:

Laptop now knows:

✔ Router MAC for IPv6 (NDP)

✔ Router MAC for IPv4 (ARP)

##  DNS Queries (Dual-stack)

IPv6 DNS Flow (RDNSS)

18. {UDP/53 DNS Query → IPv6 DNS GUA}

Src: Laptop GUA

Dst: DNS IPv6 GUA

Contents:

Query A (IPv4 address of website)

Query AAAA (IPv6 address)

19. {DNS Reply From IPv6 DNS}

Returns:

A record

AAAA record

Laptop now knows IPv6 + IPv4 addresses of website.

IPv4 DNS Flow (Local router DNS relay)

20. {ARP Request → DNS IPv4}

Laptop ARPs to reach DNS IPv4.

21. {ARP Reply from DNS/Router}

DNS MAC learned.

22. {DNS Query via IPv4}

Src: Laptop IPv4

Dst: Router IPv4 (DNS relay)

Queries: A + AAAA

23. {DNS Response via IPv4}

Router replies with A + AAAA.

📌 Phase summary:

Laptop now has:

✔ 2× AAAA (IPv6 addresses of website)

✔ 2× A (IPv4 addresses of website)

✔ Ready to start Happy Eyeballs

## Happy Eyeballs Connection Attempts

24. {TCP SYN over IPv6}

Src IP: Laptop GUA

Dst IP: Website IPv6

Src MAC: laptop

Dst MAC: router MAC

Path: Laptop → Router → ISP → Website

25. Parallel attempt (after ~200–300ms) — {TCP SYN over IPv4}

Src IP: Laptop IPv4

Dst IP: Website IPv4

NAT on router:

Laptop 192.168.1.x → Router public IPv4

26. First SYN/ACK received wins

If IPv6 SYN/ACK arrives:

{TCP SYN/ACK via IPv6}

Laptop → ACK → connection established.

Else fallback:

{TCP SYN/ACK via IPv4}

📌 Phase summary:

Laptop selects best path:

✔ Prefer IPv6 if fast

✔ Use IPv4 if IPv6 slow/unreachable

(Happy Eyeballs RFC 8305)

## TLS Handshake + HTTPS

27. TLS ClientHello
  
29. ServerHello + Certificate
    
31. Client Finished
    
33. Encrypted HTTP/3 or HTTP/2 or HTTP/1.1 begins
