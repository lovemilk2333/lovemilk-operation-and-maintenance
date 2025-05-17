# NFS 网络文件系统配置
# 配置 NFS 服务端
> 注意  
> 原版的 NFS 不支持 Auth 鉴权, 无法使用密码保护文件夹  
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

# 进阶: NFS Kerberos 鉴权
> Kerberos 与 NFS 结合使用, 可为 NFS 添加一层额外的安全保护  
> 它可以是一种更强大的身份验证机制, 也可以用于对 NFS 流量进行签名和加密
<!-- TODO -->

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
