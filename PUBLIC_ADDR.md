# Public IP Addresses

By default, NAT is used and [RFC 1918](https://tools.ietf.org/html/rfc1918) 
addresses are assigned to the clients for IPv4 and 
[RFC 4193](https://tools.ietf.org/html/rfc4193) addresses for IPv6.

In case you want to use routable IPv4 and/or IPv6 addresses you need to 
modify the configuration and make sure your upstream router routes the 
appropriate range(s) to your VPN server's IP address(es).

First edit `/etc/vpn-user-portal/config.php` and set the `xRangeFour` and/or 
`xRangeSix` addresses as appropriate, where `x` is either `w` or `o` depending
on whether you are configuring WireGuard or OpenVPN.

Next, modify the firewall according to 
[these](FIREWALL.md#public-ip-addresses-for-vpn-clients) instructions.

To apply the configuration changes:

```
$ sudo vpn-maint-apply-changes
```
