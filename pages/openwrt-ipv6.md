# OpenWRT IPv6 的三种写法

## relay6

这是推荐方案。

首先，清空 IPv6 ULA Prefix，两种方法：

一是在 luci 网页的 `Network - Interfaces - Global network options` 里，清空然后应用。

二是在终端里执行：

```bash
uci set network.globals.ula_prefix=""
uci commit network
```

然后，在终端里执行：

```bash
uci set dhcp.lan.ra_management='0'
uci set dhcp.lan.ra='relay'
uci set dhcp.lan.ra_default='0'
uci set dhcp.lan.dhcpv6='relay'
uci set dhcp.lan.ndp='relay'
uci set dhcp.lan.master='1'
uci commit dhcp
```

这其实相当于编辑 `/etc/config/dhcp` 文件，在 `config dhcp 'lan'` 节做了如下配置：

```
config dhcp 'lan'
    ...
    option ra 'relay'
    option dhcpv6 'relay'
    option ndp 'relay'
    option master '1'
```

这四句里的 `option master '1'` 在 luci 网页里没有设置的方法，所以只能用 uci 或编辑文件。

如果是直接编辑的文件，那么保存后还需要重启网络：

```bash
/etc/init.d/network restart
```

这么设置后，路由器将中继 IPv6 的 RA DHCPv6 NDP，局域网内的设备能够直接从上级路由器获取到 IPv6 地址。

## nat6

本节参考了[官方文档][1]。

因为 luci 尚不支持配置 IPv6 NAT，所以部分需要终端操作，下面全程用终端演示。

```bash
# 先 SSH 到路由器，这里略去不表

# 本质上需要安装的是 kmod-ipt-nat6
opkg update
opkg install ip6tables-mod-nat

# ULA 即我们要配置的内网，选一个你喜欢的 /48 以内的子网段
uci set network.globals.ula_prefix="fc00:2333::/48"
uci commit network

# set to DHCPv6 server mode
uci set dhcp.lan.ra_management='1'
uci set dhcp.lan.ra='server'
uci set dhcp.lan.ra_default='1'
uci set dhcp.lan.dhcpv6='server'
uci set dhcp.lan.ndp='disabled'
uci commit dhcp

# NAT6 最关键的就是 masquerading
zone_id=$(uci show firewall | sed -ne "s/^firewall\.@zone\[\(\d\+\)\]\.name='wan'\$/\1/p")
uci set firewall.@zone[${zone_id:?}].masq6=1
# 这里配置优先选用 static address (0) 还是 temporary address (1)
uci set firewall.@zone[${zone_id:?}].masq6_privacy=0

# NAT6 不需要 ICMPv6 转发
rule_id=$(uci show firewall | grep 'Allow-ICMPv6-Forward' | cut -d'[' -f2 | cut -d']' -f1)
uci set firewall.@rule["${rule_id}"].enabled='0'

uci commit
```

因为现阶段的 ip6tables-mod-nat 做的不好，我们还要编辑 `/etc/firewall.user` 追加一些内容：

```bash
# 请改成自己的 WAN 接口
WAN_IF=eth0.2
ip6tables -t nat -A POSTROUTING -o $WAN_IF -j MASQUERADE
```

上面这步实际上表明了我们用 uci 给 `/etc/config/firewall` 里的 `zone lan` 配置 `option masq6 '1'` 没有什么作用。但是，注意！**Routing/NAT flow offloading 会使用这个值**，因此乖乖配好它，这样我们能够开启硬件 NAT 转发，提高路由器性能。

如无意外，在这时我们已经配好了 nat6。

但是果壳的校园网就是个意外。我们发现，来自上游的默认路由长这个样子：

```bash
# ip -6 route
default from 2001:cc0:2020:3020::/64 via fe80::xxxx dev eth0.2  metric 512
```

这么优秀的默认路由，我们的 ULA 子网没法用啊！没匹配到这条路由，因此 LAN 的数据包不会走 WAN 口，直接就 Host unreachable 了。

在我的使用过程中还发现，其实上游存在多个路由器，而上游通告的默认路由不一定能通。即上面的 `fe80::xxxx` 不一定是能通的那个链路。无论如何，请打开终端，登上路由器，运行以下命令：

```bash
# ip -6 neigh show dev $WAN_IF | grep router
fe80::xxxx lladdr xmac router used 3/2/1 probes 1 STALE
fe80::yyyy lladdr ymac router used 3/1/3 probes 1 REACHABLE
```

如果你像我一样找到了多个邻居，不知道哪条能用？在 luci - network - static routes 里都加进去：

```
interface  target    gateway     metric  mtu      route type
wan6       2000::/3  fd80::xxxx  256     [blank]  unicast
wan6       2000::/3  fd80::yyyy  65535   [blank]  unicast
```

上表里 metric 是选用优先级，默认值是 1024，越小越优先。

上表里指定为 256 的因为较小，会被优先选用，而 65535 的就差不多完全不会选用。把某一条路由项的 metric 设为 256，其他的设为 65535，保存并应用后打开[一个纯 IPv6 网站][2]试试，不行就换下一个。

## bridge6

这是一个在链路层的解决方案，部分路由器可以正常工作。

```bash
BR_IF=br-lan
WAN_IF=wan0.2

# 非 IPv6 数据包不走网桥，直接走 IP 层路由
ebtables -t broute -A BROUTING -i $BR_IF -p ! ipv6 -j DROP

# 把 WAN 连入网桥
brctl addif $BR_IF $WAN_IF
```

如果路由器也需要 IPv6，那么可以在 `$LAN_IF` 上打开 DHCPv6 Client，但要注意关闭 RA。

[1]: https://openwrt.org/docs/guide-user/network/ipv6/ipv6.nat6
[2]: https://mirrors.tuna.tsinghua.edu.cn/
