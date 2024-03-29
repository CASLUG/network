# 网络现状

- 中国科学院（CAS）使用的是中国科技网（CSTNET）[1][1] [2][2]。
- 中国科学院大学（UCAS）使用的是中国教育网（CERNET）[1][3]。
- UCAS 雁栖湖东区提供中国移动的 CMCC-EDU 无线网，西区提供中国联通的 ChinaUnicom 无线网。
- UCAS 中关村部分宿舍楼（F）有 CMCC-EDU，部分宿舍楼（G）有 ChinaUnicom。

注：国内 ISP 的 ASN 可在[这里][4]查询。

注：联通 ChinaUnicom 热点现阶段是免费的，不计流量不计时，但需要手动申请开通。

[1]: https://ipinfo.io/AS17965
[2]: https://ipinfo.io/AS37944
[3]: https://ipinfo.io/AS4538
[4]: https://ipinfo.io/countries/cn

## 网络带宽

- 雁栖湖校区宿舍网口 IPv6 可能为千兆上下行对等。
- //待补充

## 网址分配

- UCAS 中关村宿舍楼分配的 IPv6 使用 DHCPv6，无 PD 前缀。[^注1]
- UCAS 雁栖湖宿舍楼墙口和无线网分配的 IPv6 使用 DHCPv6，无 PD 前缀；分配的 IPv4 为公网地址。[^注2]
- UCAS 雁栖湖教学楼、学园楼、图书馆的无线网有一定概率分配到 IPv6 地址。分配的 IPv4 为私网地址。[^注3]
- UCAS 雁栖湖西区宿舍可连到 `ChinaUnicom`，登录后分配到公网 IPv4 地址。[^注4]
- ISCAS 分配的 IPv6 使用 DHCPv6，无 PD 前缀。[^注5]
- //待补充

[^注1]: F 座分配的 IPv6 地址在 `2001:cc0:2020:3020::/64` 子网下，且主机地址是根据网卡 MAC 生成的 EUI-64 IID。  
[^注2]: 暂未发现防火墙有屏蔽 80 端口的情况。  
[^注3]: 分配的 IPv4 地址都在 `10.0.0.0/8` 下，不同位置的具体网段不同，每个子网为 `/23` 大小。连接学校无线网时，无需认证登录即可连通宿舍楼的 IP。  
[^注4]: 暂未发现防火墙屏蔽端口的情况。  
[^注5]: 因为各实验室自行管理内网，不是所有实验室都支持 IPv6。  
