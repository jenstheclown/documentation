# Profile Configuration

Profiles, are configured in `/etc/vpn-user-portal/config.php` and
can contain many options to support various deployment scenarios. These are 
described in the table below.

The configuration file itself also contains explanations of all options and 
their (default) value. This document has some additional explanation and 
discusses some considerations.

To modify any of the options, modify the file mentioned above and look for the
`ProfileList` section, e.g:

```
'ProfileList' => [
    [
        'profileId' => 'default',
        'displayName' => 'Default',
        ...
        ...
    ],
],
```

Every profile has an identifier (`profileId`) in this case `default`. It must 
be unique.

On Fedora you may also need to take a look at the [SELinux](SELINUX.md) 
instructions.

## Options

This table describes all available profile configuration options. The 
"Default" column indicates what the value is if the option is _missing_ from 
the configuration.

Be careful when changing configuration options. They *MAY* break existing 
VPN client connections when not using the native eduVPN / Let's Connect! 
applications.

### Common

Common configuration options, independent of the VPN protocol. See 
[WireGuard](#wireguard) and [OpenVPN](#openvpn) for protocol specific options.

| Option                                         | Type                   | Default                  |
| ---------------------------------------------- | ---------------------- | ------------------------ |
| [profileId](#profile-id)                       | `string`               | _N/A_                    |
| [displayName](#display-name)                   | `string`               | _N/A_                    |
| [hostName](#host-name)                         | `string[]` or `string` | _N/A_                    |
| [defaultGateway](#default-gateway)             | `bool`                 | `true`                   |
| [dnsServerList](#dns-server-list)              | `string[]`             | `[]`                     |
| [routeList](#route-list)                       | `string[]`             | `[]`                     |
| [excludeRouteList](#exclude-route-list)        | `string[]`             | `[]`                     |
| [aclPermissionList](#acl-permission-list)      | `string[]`             | `null`                   |
| [dnsSearchDomainList](#dns-search-domain-list) | `string[]`             | `[]`                     |
| [nodeUrl](#node-url)                           | `string[]` or `string` | `http://localhost:41194` |
| [onNode](#on-node)                             | `int[]` or `int`       | `0`                      |
| [preferredProto](#preferred-protocol)          | `string`               | `openvpn`                |

#### Profile ID

The profile ID is used to uniquely identify a profile. It can only contain 
letters, `[a-z]`, numbers `[0-9]` and the dash (`-`). Examples of valid profile 
IDs identifiers are `employees`, `students`, `admin`. It MUST NOT be numeric,
e.g. `1`, or `214`.

#### Display Name

Specify the name you want to give this profile. This will be visible to the
users and allows them to determine which protocol to select if more than one
is available.

#### Host Name

This is the DNS name that VPN clients will use to connect to the VPN 
server. It is _highly recommended_ that you make the profile ID part of the 
DNS name to allow for effective load balancing, or moving profiles to different
servers. As an example, if your profile has the name `employees`, your DNS name
could be `employees.vpn.example.org`. Obviously, you can make this a `CNAME` to
`vpn.example.org` as long as you have only one server.

#### Default Gateway

This option allows you to indicate to the VPN client that all their traffic
needs to be sent over the VPN.

#### DNS Server List

Provide a list of DNS servers to your VPN clients, as an example:

```
'dnsServerList' => ['9.9.9.9', '2620:fe::9'],
```

The DNS server list is provided to the VPN clients when either of the following
conditions hold:

1. [Default Gateway](#default-gateway) is set;
2. Default Gateway is _not_ set, but 
   [DNS Search Domain List](#dns-search-domain-list) _is_.

**NOTE**: when [Default Gateway](#default-gateway) is set and DNS servers are 
configured, on Windows, traffic to DNS servers outside the VPN will be 
explicitly blocked in order to prevent 
[DNS leaks](https://en.wikipedia.org/wiki/DNS_leak).

**NOTE**: make sure the DNS server(s) you provide are reachable by the clients,
so you MAY have to add the addresses to [Route List](#route-list) when they are
not usable from outside the VPN.

**NOTE**: when [Default Gateway](#default-gateway) is set and DNS servers are
configured the [DNS Search Domain List](#dns-search-domain-list) is ignored, as 
all DNS queries will go to the configured DNS servers anyway.

**NOTE**: with vpn-user-portal >= 3.1.6 you can use the placeholder values 
`@GW4@` and `@GW6@` that will be replaced by the address of the VPN gateway. 
This is especially helpful if you have multiple OpenVPN processes in one 
profile that want to use the DNS resolver running on the VPN server. This 
restores behavior that was available in the 2.x server. In order to configure
this see [Local DNS](LOCAL_DNS.md).

##### Public DNS Providers

If you are looking for public DNS providers, we are aware of the following 
ones. We can't vouch for any of them, obviously.

* [dns0.eu](https://www.dns0.eu/) (Strictly EU based, claims GDPR compliancy)
* [Quad9](https://www.quad9.com/)
* [Cloudflare](https://1.1.1.1/dns/)
* [Google Public DNS](https://developers.google.com/speed/public-dns/)
* [DNS.WATCH](https://dns.watch/)

#### Route List

If you are _not_ using the VPN as a [Default Gateway](#default-gateway) you can
specify the _routes_ for which the VPN client needs to use the VPN. You use 
this in the scenario that some internal services need to be reachable by your
VPN clients, but you do not want all of the client's traffic over the VPN.

Example:

```
'routeList' => ['10.223.140.0/24', 'fc5b:7c64:3001:a95f::/64'],
```

#### Exclude Route List

This is the opposite of [Route List](#route-list). Here you list all prefixes
that are _not_ supposed to go over the VPN. There are two uses cases for this:

1. Route all traffic over VPN, i.e. [Default Gateway](#default-gateway), 
   _except_ the prefixes listed here. This can be used to e.g. not send traffic
   to a "cloud" video conferencing solution over the VPN, but send it direct;
2. Do _not_ send traffic to a subset of the prefixes sent using 
   [Route List](#route-list) over the VPN.
   
Example: send all traffic over the VPN, _except_ traffic to Quad9's DNS:

```
'defaultGateway' => true,
'excludeRouteList` => ['9.9.9.9/32', '149.112.112.9/32', '2620:fe::9/128', '2620:fe::fe:9/128'],
```

As another, somewhat contrived, example that makes clear how it would work:

```
'defaultGateway' => false,
'routeList' => ['10.223.140.0/24', 'fc5b:7c64:3001:a95f::/64'],
'excludeRouteList' => ['10.223.140.5/32', 'fc5b:7c64:3001:a95f::1234/128'],
```

As of 2022-07-20 this does NOT work everywhere. It works only without issues
on Windows (both OpenVPN and WireGuard). It does NOT work properly on the rest
of the OSes:

* [--route-ipv6 does not recognize net_gateway or net_gateway-ipv6](https://community.openvpn.net/openvpn/ticket/1161) (OpenVPN)
* [Add net_gateway / net_gateway_ipv6 gateway flag for --route and --route-ipv6](https://github.com/passepartoutvpn/tunnelkit/issues/225) (OpenVPN)
* ["excluded routes" does not work](https://github.com/eduvpn/apple/issues/475) (WireGuard)

#### ACL Permission List

Restrict access to VPN profiles based on user permissions. The authentication 
module can make permissions available either through LDAP or SAML that can be
used to restrict access to a profile. See [ACL](ACL.md) for extensive 
documentation on the topic.

#### DNS Search Domain List

Allow you to specify the "Connection-specific DNS Suffix Search List" for the
VPN client, e.g.:

```
'dnsSearchDomainList' => ['example.org', 'example.com'],
```

**NOTE**: the search domains are ONLY used when DNS servers are specified and
the [Default Gateway](#default-gateway) is _not_ set.

#### Node URL

When using a separate system to handle VPN connections, i.e. when using a 
controller + node(s) setup. See [Multi Node](MULTI_NODE.md) for extensive 
documentation on the topic.

#### On Node

When deploying profile(s) to multiple nodes you may want to indicate to which
specific node(s) the profile belongs. For example, if you have 4 nodes and you
want to deploy a profile only to node 2 and 3, use the following:

```
'onNode' => [2, 3],
```

If you do NOT specify the `onNode` option, it will default to `0` if there is
only one node defined, or if there are multiple nodes start counting from 0, 
i.e. if you have two nodes defined, the default will be `[0, 1]`.

This option is available from vpn-user-portal >= 3.0.6. It was added due to a 
[bug](https://todo.sr.ht/~eduvpn/server/90) that did not allow deploying 
profiles to only a subset of nodes. Unfortunately we can not break existing
configurations, so we needed to introduce a new option specifically for this
case.

#### Preferred Protocol

When your profile supports multiple protocols, this option can be used to set
the preferred protocol. This allows for example to transitioning (the majority 
of users) from OpenVPN to WireGuard without breaking existing configurations.

The preferred protocol only makes sense when both OpenVPN and WireGuard are
enabled. If only one protocol is enabled, that is automatically the preferred 
protocol.

OpenVPN is considered enabled when both `oRangeFour` and `oRangeSix` are set. 
WireGuard is considered enabled when both `wRangeFour` and `wRangeSix` are set.


### WireGuard

WireGuard specific configuration options.

| Option                              | Type                   | Default |
| ----------------------------------- | ---------------------- | ------- |
| [wRangeFour](#wireguard-range-four) | `string[]` or `string` | _N/A_   |
| [wRangeSix](#wireguard-range-six)   | `string[]` or `string` | _N/A_   |

We wrote some additional documentation on WireGuard [here](WIREGUARD.md).

#### WireGuard Range Four

Specify the IPv4 range for WireGuard VPN clients. As an example:

```
'wRangeFour' => '172.24.110.0/24',
```

**NOTE**: make sure the specified range is unique, and not used by any other
profile/protocol, nor overlap the range specified in another profile/protocol!

**NOTE**: the UDP port used by WireGuard is `51820` and can be modified by 
changing the `listenPort` option under the `WireGuard` section in 
`/etc/vpn-user-portal/config.php`. This port is "global" and NOT per profile.

#### WireGuard Range Six

Specify the IPv6 range for WireGuard VPN clients. As an example:

```
'wRangeSix' => 'fd99:ede1:b56d:f19e::/64',
```

**NOTE**: make sure the specified range is unique, and not used by any other
profile/protocol, nor overlap the range specified in another profile/protocol!

### OpenVPN

OpenVPN specific configuration options.

| Option                                            | Type                   | Default  | Notes                            |
| ------------------------------------------------- | ---------------------- | -------- | -------------------------------- |
| [oRangeFour](#openvpn-range-four)                 | `string[]` or `string` | _N/A_    |                                  |
| [oRangeSix](#openvpn-range-six)                   | `string[]` or `string` | _N/A_    |                                  |
| [oBlockLan](#openvpn-block-lan)                   | `bool`                 | `false`  |                                  |
| [oEnableLog](#openvpn-enable-log)                 | `bool`                 | `false`  |                                  |
| [oUdpPortList](#openvpn-port-list)                | `int[]`                | `[1194]` |                                  |
| [oTcpPortList](#openvpn-port-list)                | `int[]`                | `[1194]` |                                  |
| [oExposedUdpPortList](#openvpn-exposed-port-list) | `int[]`                | `[]`     |                                  |
| [oExposedTcpPortList](#openvpn-exposed-port-list) | `int[]`                | `[]`     |                                  |
| [oListenOn](#openvpn-listen-address)              | `string[]` or `string` | `::`     | Also accepts `string[]` >= 3.0.1 |

#### OpenVPN Range Four

Specify the IPv4 range for OpenVPN VPN clients. As an example:

```
'oRangeFour' => '172.18.131.0/24',
```

**NOTE**: make sure the specified range is unique, and not used by any other
profile/protocol, nor overlap the range specified in another profile/protocol!

#### OpenVPN Range Six

Specify the IPv6 range for OpenVPN VPN clients. As an example:

```
'oRangeSix' => 'fdb4:2da1:2f15:a488::/64',
```

**NOTE**: make sure the specified range is unique, and not used by any other
profile/protocol, nor overlap the range specified in another profile/protocol!

#### OpenVPN Block LAN

This OpenVPN only option prevents the client from accessing devices on the 
local network. This is especially useful when the client is connected to a 
network that also contains clients that are not to be trusted, e.g. (semi) 
public WiFi.

```
'oBlockLan' => true,
```

#### OpenVPN Enable Log

This OpenVPN only option enables OpenVPN server logging. This can be used to
debug (some) connection issues with incompatible clients.

```
'oEnableLog' => true,
```

Once logging is enabled (and changes applied), you can follow the log like 
this:

```bash
$ sudo journalctl -f -t openvpn
```

**NOTE**: this option should probably only be enabled on test systems and not 
in production.

#### OpenVPN Port List

List of UDP/TCP ports to be used by the OpenVPN processes. The IP ranges 
[OpenVPN Range Four](#openvpn-range-four) and 
[OpenVPN Range Six](#openvpn-range-six) will be evenly distributed over both 
UDP Port List and TCP Port List.

```
'oUdpPortList' => [1194, 1195, 1196],
'oTcpPortList' => [1194],
```

Assuming your `oRangeFour` is `172.18.131.0/24` and your `oRangeSix` is 
`fdb4:2da1:2f15:a488::/64`, four processes will be created that with the 
following IP ranges:

| Process    | Range Four          | Range Six                      | # Clients |
| ---------- | ------------------- | ------------------------------ | --------- |
| `udp/1194` | `172.18.131.0/26`   | `fdb4:2da1:2f15:a488::/112`    | 61        |
| `udp/1195` | `172.18.131.64/26`  | `fdb4:2da1:2f15:a488::1:0/112` | 61        |
| `udp/1196` | `172.18.131.128/26` | `fdb4:2da1:2f15:a488::2:0/112` | 61        |
| `tcp/1194` | `172.18.131.192/26` | `fdb4:2da1:2f15:a488::3:0/112` | 61        |

**NOTE**: depending on the expected VPN usage, you should aim for a `/25` or 
`/26` for every process.

**NOTE**: the provided `oRangeSix` is always split in networks of size `/112`, 
the "smallest" network supported by OpenVPN.

**NOTE**: the total number of ports combining `oUdpPortList` and `oTcpPortList` 
specified MUST be a power of 2, e.g. 1, 2, 4, 8, 16, 32 or 64.

**NOTE**: you should aim for around 75% UDP processes, and 25% TCP processes 
for optimal performance for most clients.

See also: [Port Sharing](PORT_SHARING.md), [Multi Profile](MULTI_PROFILE.md).

#### OpenVPN Exposed Port List

TBD.

#### OpenVPN Listen Address

You can configure the OpenVPN processes to listen on a specific IPv4 _or_ IPv6 
address. This MAY be helpful in certain network configurations where a proper
configuration using [Source Routing](SOURCE_ROUTING.md) is not possible. 

By using this option your VPN clients lose the ability to connect over IPv4 
_or_ IPv6 support. The default is `::` which allows connecting over both IPv4 
and IPv6.

Example:

```
'oListenOn' => '10.5.5.7',
```

**NOTE**: using this option is NOT recommended and should be avoided if 
possible.

**NOTE**: you can also specify an array of IP addresses when running >= 3.0.1, 
this allows you to specify an IP per node when using multiple nodes.

## Apply Changes

To apply the configuration changes:

```bash
$ sudo vpn-maint-apply-changes
```
