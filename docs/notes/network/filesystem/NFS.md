# NFS 网络文件系统配置
# 配置 NFS 服务端
> [!WARNING]
> 警告  
> 原版的 NFS 不支持鉴权, 无法使用密码保护文件夹  
> 这可能带来安全性问题, 切勿部署至公共网络; 必要时请配置防火墙
>  
> 要使用带有密码验证或密钥对验证的 NFS, 请参阅 [进阶: NFS 服务的 Auth 鉴权](#进阶-nfs-kerberos-鉴权)

## 安装
安装 NFS 服务端

Debian / Ubuntu
```sh
sudo apt install nfs-kernel-server
```
ArchLinux
```sh
sudo pacman -S nfs-utils
```
> 其他 Linux 系统请安装 `nfs-utils` 软件包

启动 NFS 服务

Systemd
```sh
sudo systemctl start nfs-kernel-server.service
```

## 基础配置
修改配置文件 `/etc/exports`
```conf
<path>     <host1>(<option1>,<option2>,<...options>) <host2>(<option1>,<option2>,<...options>) <...hosts>(<option1>,<option2>,<...options>)
```

例如, 要分享 `/nfs-shared` 文件夹 (读写) 给 `192.168.1.1/24` 的客户端, 可以这样配置
```conf
/nfs-shared    192.168.1.1/24(rw,async)
```

更新配置
```sh
sudo exportfs -a
```

### 常用 `option`
> [!NOTE]
> 若无特殊说明, 同子选项的配置互斥

| 配置 | 描述 | (是)默认值 |
| :-: | :- | :-: |
| | `ro, rw` | 必填 |
| `ro` | Read Only 只读 |  |
| `rw` | Read (and) Write 可读写 |  |
| | `async, sync` | |
| `async` | 异步写入, 先缓存内存 | `nfs-utils` <= 1.0.0 |
| `sync` | 同步写入, 待上次写入完成后再次写入 (可同时发生多个写入请求, 但是会等待上次写入请求完成) | `nfs-utils` > 1.0.0 |
| | `wdelay, no_wdelay` (限 `sync`) | |
| `wdelay` | 将多个短时间内的写入请求合并 | 是 |
| `no_wdelay` | 关闭 `wdelay`, 所有写入立即独立执行 |  |
| | `subtree_check` | 部分服务必填 |
| `no_subtree_check` | (可能会造成安全隐患, 但是可能可以略微提升性能) 禁用 `subtree_check`, 当分享的文件夹不是某个文件系统的根时, 需要先判断父文件系统的权限等信息, 禁用后可以提升性能 | `nfs-utils` >= 1.1.0 (`subtree_checking` 往往会带来更多问题) |
| `subtree_checking` | 启用 `subtree_check` | `nfs-utils` < 1.1.0 |
| | `secure, insecure` | |
| `secure` | 要求客户端请求的目标端口 < `IPPORT_RESERVED` (1024) | 是 |
| `insecure` | 关闭 `secure` 的限制 | |
| | `squash` | |
| `root_squash` | 将 `uid/gid 0` 的请求映射到匿名 `uid/gid` (`nfsnobody`) | |
| `no_root_squash` | 关闭 `root_squash` | |
| `all_squash` | 将所有请求映射到匿名 `uid/gid` (`nfsnobody`) | 是 |
| | - |
| `anonuid` | 匿名 `uid` (`nfsnobody`) 的服务端本地映射用户 `uid` | `nfsnobody` |
| `anongid` | 匿名 `gid` (`nfsnobody`)  的服务端本地映射用户 `gid` | `nfsnobody` |
| `sec` | 认证方式, 参见[在 NFS 中使用 Kerberos](#在-nfs-中使用-kerberos) | `sys` (不使用 Kerberos) |

> 查看完整配置, 请参阅 [exports(5): NFS server export table - Linux man page](https://linux.die.net/man/5/exports)

# 配置 NFS 客户端
## 安装
```sh
sudo apt install nfs-common
```

## 挂载
```sh
sudo mount <host>:<path> <target-path>
```
示例: 将 `192.168.1.1:/nfs-shared` 挂载到 `/mnt/nfs-shared`
```sh
sudo mount 192.168.1.1:/nfs-shared /mnt/nfs-shared
```
> [!WARNING]
> 警告  
> 挂载点 `target-path` 必须为已存在的文件夹, 且必须为空, 否则在 nfs 文件系统卸载之前, 这些文件或子目录将无法访问

要自动挂载 NFS, 请写入 `/etc/fstab`
```fstab
<host>:<path> <target-path> nfs <option1>,<option2>,<...options>
```
示例: 将 `192.168.1.1:/nfs-shared` 挂载到 `/mnt/nfs-shared`
```fstab
192.168.1.1:/nfs-shared /mnt/nfs-shared nfs rw,rsize=8192,wsize=8192,timeo=14,intr
```

## 常见 `option`
| 配置 | 描述 | (是)默认值 |
| :-: | :- | :-: |
| `rw`, `ro` | 读写/只读 | `rw` |
| `rsize` | 每次从服务器读取的数据块的大小 (Bytes), 必须为 `1024` 正整数倍 |  |
| `wsize` | 每次向服务器写入的数据块的大小 (Bytes), 必须为 `1024` 正整数倍 |  |
| `timeo` | 超时时间 (* 1/10秒), `14` 代表 1.4秒 | |
| `intr` | 允许挂载中断 | |

> 查看完整配置, 请参阅 [Ubuntu Manpage: nfs - fstab format and options for the nfs file systems](https://manpages.ubuntu.com/manpages/trusty/man5/nfs.5.html)

# 进阶: NFS Kerberos 鉴权 (暂未尝试成功, 酌情使用)
> Kerberos 与 NFS 结合使用, 可为 NFS 添加一层额外的安全保护  
> 它可以是一种更强大的身份验证机制, 也可以用于对 NFS 流量进行签名和加密

## 安装
安装 krb5-user (客户端和服务端), krb5-kdc krb5-admin-server (服务端)

Debian / Ubuntu
```sh
sudo apt install krb5-user krb5-kdc krb5-admin-server
```
ArchLinux (`krb5` 包含客户端和服务端)
```sh
sudo pacman -S krb5
```

## 服务端配置
## 初始配置
1. 创建域
修改 `/etc/krb5.conf`, 在 `realms` 中添加添加指定的域
```conf
# 可选, 配置默认域
[libdefaults]
    default_realm = EXAMPLE.COM

# 配置域
[realms]
    EXAMPLE.COM = {
        # 配置 `kdc` (认证信息发放服务)
        # 在 DNS 存在 SRV 记录的情况下是可选的
        kdc = kdc.example.com
        # 配置 `admin_server` (管理信息发放服务)
        admin_server = kdc.example.com
        # 用户必须在请求 TGT（初始票据）时先提供密码进行预认证, 可以提升安全性
        # 使用 `-preauth` 以禁用 (须要 Kerberos >= 5, Kerberos v4 不受支持)
        default_principal_flags = +preauth
    }

# 配置域名与域的映射/匹配
[domain_realm]
    # 使用 `.domain` 代表以 domain 结尾的任意(层级)子域
    # 例如 `.example.com` 不含 `example.com`,
    # 但是包括 `a.example.com`, `b.a.example.com`, `....example.com`

    # 此时, 当客户端提供了 `*.example.com` 时, 将匹配 `[realms]` 中的 `EXAMPLE.COM` 配置
    .example.com = EXAMPLE.COM
    example.com = EXAMPLE.COM

# 可选, 日志配置
[logging]
    # 可以是 `SYSLOG:<level>` 写入系统日志,
    # 也可以直接跟文件例如 `/var/log/krb5kdc.log`
    kdc          = SYSLOG:NOTICE
    admin_server = SYSLOG:NOTICE
    default      = SYSLOG:NOTICE
```

其中, 可选的系统日志级别为
| 级别    | 描述   |
| :-:     | :-:    |
| `EMERG`   | 紧急   |
| `ALERT`   | 警报   |
| `CRIT`    | 严重   |
| `ERR`     | 错误   |
| `WARNING` | 警告   |
| `NOTICE`  | 注意   |
| `INFO`    | 信息   |
| `DEBUG`   | 调试   |

2. 创建数据库
```sh
sudo kdb5_util -r <realm> create -s
```
例如
```sh
sudo kdb5_util -r EXAMPLE.COM create -s
```
并输入数据库主密码 (输入过程不会显示, 直接回车以使用空密码)
```
Enter KDC database master key: 
```

3. 启用 (开机自启) 并 启动 Kerberos 服务
Ubuntu / Debian
```sh
sudo systemctl enable --now krb5-kdc krb5-admin-server
```

ArchLinux
```sh
sudo systemctl enable --now krb5-kdc krb5-kadmind
```

> 个人遇到了 `没有那个文件或目录 while opening ACL file /etc/krb5kdc/kadm5.acl` 的问题, 可以尝试手动创建 `/etc/krb5kdc/kadm5.acl`
> 
> ```sh
> sudo vim /etc/krb5kdc/kadm5.acl
> ```
> 写入基础内容 (`EXAMPLE.COM` 替换为您的 `realm`)
> ```conf
> */admin@EXAMPLE.COM    *
> ```
> 修改权限 (若需要)
> ```
> sudo chown root:root /etc/krb5kdc/kadm5.acl
> sudo chmod 644 /etc/krb5kdc/kadm5.acl
> ```
> 重启 `krb5-admin-server`
> ```sh
> sudo systemctl restart krb5-admin-server
> ```

### 主体配置
主体是 Kerberos 的最小可认证单元, 类似于 用户

1. 添加主体
使用本地身份验证启动 Kerberos 管理工具
```sh
sudo kadmin.local
```
运行后, 将进入一个交互式环境, 可以在内输入 Kerbero 命令, `q` / `quit` / `exit` 退出

- 用户主体:  
    使用 `addprinc` 命令添加主体

    ```sh
    addprinc <user>[@<realm>]
    ```
    例如使用如下命令交互式地创建 `EXAMPLE.COM` 域下的 `bob` 主体
    ```sh
    addprinc bob@EXAMPLE.COM
    ```
    或使用如下命令直接指定密码 (生产环境不推荐, 可能有 SHELL 历史)
    ```sh
    addprinc -pw password bob@EXAMPLE.COM
    ```

    看到 `Principal "<user>@<realm>" created.` 字样表示创建成功

- KDC主体:  
    添加 KDC 主体 (随机生成密钥)

    生成密钥
    ```sh
    addprinc -randkey <service>/<host-name>
    ```

    添加 KDC 密钥
    ```sh
    ktadd <service>/<host-name>
    ```

    > 其中, `<service>` 是一个便于标识的服务名称, 一般是根据服务约定俗成的名称, 例如 NFS 为 `nfs`,  
    > `<host-name>` 是该服务的 [FQDN 主机名](https://en.wikipedia.org/wiki/Fully_qualified_domain_name), 与实际 DNS 或 `/etc/hostname` 匹配

> 使用 `delprinc` 命令删除主体  
> 使用方法与 `addprinc` 类似

### 使用/解使用用户主体
```sh
kinit [<user>[@<realm>]]
```
例如
```sh
kinit bob@EXAMPLE.COM
```
> 若配置了 `libdefaults`, 可以直接使用 `kinit <user>` 命令  
> 配置了 `libdefaults` 且当前用户名与用户主体名称相同, 直接使用 `kinit` 命令即可

清除 ticket 缓存 (类似于退出登录)
```sh
kdestroy
```

### 查看已使用的主体
```sh
klist
```
输出类似于
```sh
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: bob@EXAMPLE.COM

Valid starting       Expires              Service principal
2025-05-30T16:40:41  2025-05-31T02:40:41  krbtgt/EXAMPLE.COM@EXAMPLE.COM
        renew until 2025-05-31T16:40:40
```

### 配置 DNS 解析
> 如果每台主机手动指定了 realm, 或 hosts 文件中配置了域名解析, 则不需要配置 DNS 解析  
> 当然, 在 DNS 服务器上配置 hosts 尚可

配置如下 DNS 解析
```
kerberos.example.com.           A     <KDC-IP>
_kerberos.example.com.          TXT   <realm-name> # 例如 `EXAMPLE.COM`
_kerberos._udp.example.com.     SRV   0 0  88 kerberos.example.com.
_kerberos-adm._udp.example.com. SRV   0 0 749 kerberos.example.com.
```
> 注意  
> `kinit` 时的 hosts 仍然为 `EXAMPLE.COM`,  
> 因为 `_kerberos._udp.example.com` 标识了 KDC 服务该如何访问  
>  
> 须要注意的是: `kerberos._udp.example.com` 必须位于 `realm` 的同个子域下  
> 例如 `EXAMPLE.COM` 的 `realm` 下必须为 `kerberos._tcp.example.com` / `kerberos._udp.example.com`, 不能为其他子域 (除 手动配置客户端 `/etc/krb5.conf` 的 `realms` 外, 后者无需配置)

### 在 NFS 中使用 Kerberos
1. 添加 NFS 服务 (KDC) 主体
    ```sh
    sudo kadmin.local
    ```

    ```sh
    addprinc -randkey nfs/<host-name>
    ktadd nfs/<host-name>
    ```

    > 注意!  
    > NFS 客户端请求的 `host-name` 必须和 NFS 服务的 `host-name` 一致!!!

2. 修改 `/etc/exports` 的 `sec` 选项, 可选值为

| 配置 | 描述 |
| :-: | :- |
| `krb5` | 仅使用 `kerberos` 进行身份验证, 并以未经身份验证和未加密的方式传输数据 (局域网可选) |
| `krb5i` | 使用 `kerberos` 进行身份验证和完整性检查, 但以未加密的方式传输数据 (局域网可选) |
| `krb5p` | 使用 `kerberos` 进行身份验证和加密 (推荐) |
| `sys` | 不使用 `kerberos` (默认) |

> 也可以使用 `:` 分隔多个选项, 当靠前选项失败时, 依次尝试后面的选项  
> 例如: `krb5p:krb5` 会优先尝试 `krb5p`, 前者失败后再尝试 `krb5`

3. 重载 `/etc/exports` 配置
    ```sh
    sudo exportfs -arv
    ```

> 如果无法连接, 请确认服务端 `rpc-gssd` 服务状态
> ```sh
> sudo systemctl status rpc-gssd
> ```
> 启动
> ```sh
> sudo systemctl start rpc-gssd
> ```

> 或者, 尝试重启客户端后再试

## 客户端配置
1. 修改 `/etc/krb5.conf` 以添加服务端  
对于大多数情况, 您只需要简单地将服务端的配置文件复制至客户端即可,  
当然, 也可以仅添加 `realm`

2. 使用 `kinit` 以客户端主体身份登录 (命令用法参见 [使用/解使用用户主体](#使用解使用用户主体)), 并使用 `klist` 查看登录状态 (命令用法参见 [查看已使用的主体](#查看已使用的主体))

3. [挂载 NFS 文件系统](#挂载) 即可
> 添加 `-vv` 选项以获取详细信息  
> 可能需要 `-t nfs4` 和 `-o sec=krb5p` 或您选择的安全选项才能挂载成功

---
# 参考文献
1. [Network File System (NFS) - Ubuntu Server documentation](https://documentation.ubuntu.com/server/how-to/networking/install-nfs/index.html)  
2. [security - Can I password protect an NFS share? - Super User](https://superuser.com/questions/327984/can-i-password-protect-an-nfs-share)  
3. [NFSv4Howto - Community Help Wiki](https://help.ubuntu.com/community/NFSv4Howto) 
4. [9.4.3. Common NFS Mount Options | Reference Guide | Red Hat Enterprise Linux | 4 | Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/reference_guide/s2-nfs-client-config-options#s2-nfs-client-config-options)  
5. [21.7. The /etc/exports Configuration File | Deployment Guide | Red Hat Enterprise Linux | 5 | Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/5/html/deployment_guide/s1-nfs-server-config-exports#s1-nfs-server-config-exports)  
6. [exports(5): NFS server export table - Linux man page](https://linux.die.net/man/5/exports)  
7. [NFS网络文件系统配置-阿里云开发者社区](https://developer.aliyun.com/article/518700)  
8. [Ubuntu Manpage: nfs - fstab format and options for the nfs file systems](https://manpages.ubuntu.com/manpages/trusty/man5/nfs.5.html)
9. [Kerberos - Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/Kerberos)
