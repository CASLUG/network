# iptables 配置透明代理

透明代理是指，在设备完全不知情的情况下，路由器通过代理服务器而非普通方式发出和接收数据包。一般用于以下情况：校园网用户访问外网、国内用户访问外网。

因为校园网一般 IPv4 收费而 IPv6 畅游，所以可以通过一个 IPv4 IPv6 双线接入的服务器中转，实现免费上网。即 `You <--IPv6--> Proxy <--IPv4--> Network`。

在路由器上使用 iptables 规则我们可以轻松配置透明代理。

## TCP

```bash
iptables -t nat -N ss
iptables -t nat -F ss
iptables -t nat -A ss -d 0.0.0.0/8 -j DROP # source only
iptables -t nat -A ss -d 127.0.0.0/8 -j RETURN # loopback
iptables -t nat -A ss -d 10.0.0.0/8 -j RETURN # A-private
iptables -t nat -A ss -d 169.254.0.0/16 -j RETURN # link-local
iptables -t nat -A ss -d 172.16.0.0/12 -j RETURN # B-private
iptables -t nat -A ss -d 192.0.0.0/24 -j RETURN # reserved
iptables -t nat -A ss -d 192.0.2.0/24 -j RETURN # test
iptables -t nat -A ss -d 192.168.0.0/16 -j RETURN # C-private
iptables -t nat -A ss -d 198.51.100.0/24 -j RETURN # test
iptables -t nat -A ss -d 203.0.113.0/24 -j RETURN # test
iptables -t nat -A ss -d 224.0.0.0/4 -j RETURN # multicast
iptables -t nat -A ss -d 240.0.0.0/4 -j RETURN # reserved
iptables -t nat -A ss -d 255.255.255.255 -j RETURN # boardcast

# uncommit next two lines if ss-server is listening on a IPv4 address
SS_REMOTE=
#iptables -t nat -A ss -d $SS_REMOTE -j RETURN

# ss-local should be running on the router and listening on 0.0.0.0
SS_PORT=1088
iptables -t nat -A ss -p tcp -j REDIRECT --to-port $SS_PORT

LAN_IF=br-lan
iptables -t nat -A PREROUTING -i $LAN_IF -p tcp -j ss # prefer "prerouting_lan_rule" chain for openwrt
```

如果你想使用的 ss-local 不在路由器上，那你不仅要配置 nat DNAT，还要配置 filter ACCEPT。

## UDP

```bash
iptables -t mangle -N ss
iptables -t mangle -F ss
iptables -t mangle -A ss -d 0.0.0.0/8 -j RETURN # source only
iptables -t mangle -A ss -d 127.0.0.0/8 -j RETURN # loopback
iptables -t mangle -A ss -d 169.254.0.0/16 -j RETURN # link-local
iptables -t mangle -A ss -d 10.0.0.0/8 -j RETURN # A-private
iptables -t mangle -A ss -d 172.16.0.0/12 -j RETURN # B-private
iptables -t mangle -A ss -d 192.0.0.0/24 -j RETURN # reserved
iptables -t mangle -A ss -d 192.0.2.0/24 -j RETURN # test
iptables -t mangle -A ss -d 192.168.0.0/16 -j RETURN # C-private
iptables -t mangle -A ss -d 198.51.100.0/24 -j RETURN # test
iptables -t mangle -A ss -d 203.0.113.0/24 -j RETURN # test
iptables -t mangle -A ss -d 224.0.0.0/4 -j RETURN # multicast
iptables -t mangle -A ss -d 240.0.0.0/4 -j RETURN # reserved
iptables -t mangle -A ss -d 255.255.255.255 -j RETURN # boardcast

# uncommit next two lines if ss-server is listening on a IPv4 address
SS_REMOTE=xxxx
#iptables -t mangle -A ss -d $SS_REMOTE -j RETURN

# do TPROXY and add fwmark 0x1 to packets
# ss-local should be running on the router and listening on 127.0.0.1 / 0.0.0.0
SS_PORT=1088 # ss-local
iptables -t mangle -A ss -p udp -j TPROXY --tproxy-mark 0x1/0x1 --on-ip 127.0.0.1 --on-port $SS_PORT

# routing rule:
#   for packet with fwmark 0x1, look up table 100
#   for table 100, route to "local" through device "lo" by default
ip -4 rule del fwmark 0x1 lookup 100
ip -4 rule add fwmark 0x1 lookup 100
ip -4 route del local default dev lo table 100
ip -4 route add local default dev lo table 100

# for udp, it's better to accept all input from each ip address of ss-server, because udp conntrack doesn't work well
WAN_IF=eth0.2
iptables -A INPUT -p udp -i $WAN_IF -s $SS_REMOTE -j ACCEPT # prefer "input_wan_rule" chain for openwrt

LAN_IF=br-lan
iptables -t mangle -A PREROUTING -i $LAN_IF -p udp -j ss
```

## DNS

DNS 可能是我们需求量最大的一类 UDP 请求，而因为 DNS 的特殊性，客户端事实上只关心 DNS 响应的数据包内容，而不在意报文的目的地址，所以可以使用更强的拦截规则。

```bash
DNS_PORT=53

# REDIRECT if dns server is running on the router and listening on 0.0.0.0
iptables -t nat -A zone_lan_prerouting -p udp --dport 53 -j REDIRECT --to-port $DNS_PORT

# DNAT if dns server is running on another host
#DNS_IP=xxxx
#iptables -t nat -A zone_lan_prerouting -p udp --dport 53 -j DNAT --to $DNS_IP:$DNS_PORT
```

## ipset

TODO
