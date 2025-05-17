# DHCP 动态 IP 地址分配

DHCP 是运行于应用层, 基于 UDP, 用于客户机动态获取 IP 地址, 子网掩码, 默认网关和 DNS 信息的一种协议

## 地址池
地址池表示了每个 DHCP 可分配的 IP 地址, 一般家庭网络中为局域网地址, 参见 [RFC 1918 3. Private Address Space](https://datatracker.ietf.org/doc/html/rfc1918#section-3)

## 租期
某个 独立MAC地址 的设备, 在 DHCP 申请完成后, 使用的 IP 地址最大时长  

IP 地址在到期前需要续租
