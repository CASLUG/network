# 网络现状

中国科学院（CAS）使用的是中国科技网（CSTNET）[1][1] [2][2]，中国科学院大学（UCAS）使用的是中国教育网（CERNET）[1][3]。

注：国内 ISP 的 ASN 可在[这里][4]查询。  

## 网络带宽

TODO

## 网址分配

- UCAS 中关村宿舍分配的 IPv6 使用 DHCPv6，无 PD 前缀
- ISCAS 分配的 IPv6 使用 DHCPv6，无 PD 前缀
- //待补充

在 UCAS 中关村宿舍 F 座得到的 IP 在 `2001:cc0:2020:3020::/64` 子网下，且主机地址是根据网卡 MAC 生成的 EUI-64 IID。

注：在 ISCAS，因为各实验室自行管理内网，不是所有实验室都支持 IPv6。  

[1]: https://ipinfo.io/AS17965
[2]: https://ipinfo.io/AS37944
[3]: https://ipinfo.io/AS4538
[4]: https://ipinfo.io/countries/cn
