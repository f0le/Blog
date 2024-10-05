Hyperlocal DNS resolver

What does this prevent?

This prevents 

What is Unbound?

How does DNS on Linux work?


Force OpenBSD to use unbound(8) DNS resolver in DHCP client mode
  2018-03-16      106 words, 1 minutes

    dhcp
    dns
    openbsd
    unbound

By default, a DHCP client gets an IP address, a network gateway and a DNS server. That’s fine most of the time. But if you own an OpenBSD cloud instance that has to use DHCP to get online, you might not be satisfied with the domain-name-servers option provided by your DHCP server. Hopefully, OpenBSD provides an easy way to force your DNS:

# vi /etc/dhclient.conf
(...)
prepend domain-name-servers 127.0.0.1;

Since then, OpenBSD will use our DNS resolver. Which is… unbound(8)

# rcctl enable unbound
# rcctl start unbound

Note that this configuration allows to use the DNS server provided by the DHCP server as a fallback.
