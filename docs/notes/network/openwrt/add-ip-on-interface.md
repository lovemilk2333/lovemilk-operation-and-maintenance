# 给已经存在的网卡配置 (额外) IP 段

## 方法 1: 添加一个新的 `alias` 配置
/etc/config/network
```conf
config alias
        option interface 'lan'  # 已存在的网卡

        # ... 其他配置 (例如: 静态地址)
        option proto 'static'
        option ipaddr '10.0.100.1'
        option netmask '255.255.255.0'
```

> [!WARNING]
> 注意  
> 这使得 `alias` 和 `lan` 即使不在一个网段也可以通信

<!-- ## 方法 2: 添加一个新的 `interface` 配置 -->
