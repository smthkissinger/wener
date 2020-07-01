---
id: ssh
title: SSH
---

# SSH

* [sshd_config](http://man.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man5/sshd_config.5)

## Tips
* Host key `/etc/ssh/ssh_host_*`
* 将指定用户的端口转发使得外部都可见,可在 `/etc/ssh/sshd_config` 中添加,然后 `service sshd reload`
* http://quark.humbug.org.au/publications/ssh/ssh-tricks.html
* Verify that the .pem file has permissions of 0400, not 0777


__ESCAPE__

```
 ~.   - terminate connection (and any multiplexed sessions)
 ~B   - send a BREAK to the remote system
 ~C   - open a command line
 ~R   - request rekey
 ~V/v - decrease/increase verbosity (LogLevel)
 ~^Z  - suspend ssh
 ~#   - list forwarded connections
 ~&   - background ssh (when waiting for connections to terminate)
 ~?   - this message
 ~~   - send the escape character by typing it twice
(Note that escapes are only recognized immediately after newline.)

ssh> help
Commands:
      -L[bind_address:]port:host:hostport    Request local forward
      -R[bind_address:]port:host:hostport    Request remote forward
      -D[bind_address:]port                  Request dynamic forward
      -KL[bind_address:]port                 Cancel local forward
      -KR[bind_address:]port                 Cancel remote forward
      -KD[bind_address:]port                 Cancel dynamic forward
```

__常用配置__

```bash
# 配置使用的端口
Port 22

# 是否转发网关
GatewayPorts no
# 是否允许使用 root 登陆
PermitRootLogin yes
# 是否允许使用密码登陆
PasswordAuthentication yes
ChallengeResponseAuthentication yes

# 转发的端口允许外部访问
Match User dev
   GatewayPorts yes

# 可只对指定的接口对外暴露
# -R :8000:localhost:80
# GatewayPorts clientspecified

# 禁止部分用户使用 TTY
Match User player
  PermitTTY no
```

```bash
# 强制使用密码
ssh user:@example.com
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no example.com
```

## config

```bash
# 网关服务器
Host my-gate
User root
Hostname 1.2.3.4

# 通过 my-gate 链接 1.2.3.100
# busybox 的 nc 没有 -q, 需要
Host my-gate-110
User root
ProxyCommand ssh -q qc-sh-1 nc -q0 1.2.3.100 22
```

## Key

```bash
# 生成 key
ssh-keygen -t rsa -b 2048 -C "email@example.com"

# 无密码不询问
ssh-keygen -t rsa -b 2048 -f /tmp/sshkey -q -N ""
```

## Tunnel
在工作中常常需要较多的代理和转发,为每个代理和转发都进行一次 SSH 未免太过麻烦,使用 `~/.ssh/config` 可以将常用的转发一次配置

```bash
Host tunnel
Hostname my.host.com
User myUser
Compression yes
ExitOnForwardFailure yes
ForwardAgent yes
DynamicForward 8888
RemoteForward 2222 127.0.0.1:22
LocalForward 16379 myInternalRedis:6379
LocalForward 13306 myInternalMySQL:3306
```

再配合 autossh 可大大减少工作量

```bash
autossh -M 8889  -vNg tunnel > ssh.log 2>&1 &
```

## 多路复用
* https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing
* 好处
  * 减少连接时间 - 特别是机器多、ssh命令多、ack延时高的时候
  * 连接复用
* 注意
  * 连接过多可能不问题
  * 不要用来传大文件 - 直接连接会更快

```
Host *
ControlPath ~/.ssh/controlmasters/%r@%h:%p
ControlMaster auto
ControlPersist 10m
```

```bash
# 必须要手动创建目录
mkdir ~/.ssh/controlmasters

# 检测
ssh -O check myhost
# 自动启动 master
ssh myhost pwd
# 停止 master
ssh -O stop myhost

# 手动启动 master
ssh -MNn user@server
```

## 网关

```bash
ssh -t gateway ssh internal
```

```
Host internal
  ProxyCommand ssh gw nc -w 1 internal 22
```

```bash
ssh internal
```

```bash
ssh -f -nNT -R 1100:localhost:22 somehost

ssh localhost -p 1100
```

## HTTP + SSH 多路
* https://github.com/yrutschle/sslh

## FAQ
### key_load_public: No such file or directory
找不到 `~/.ssh/id_rsa.pub`，不影响使用

### 端口转发
* -L/LocalForward 本地转发
* -R/RemoteForward 远程转发

```bash
# 访问本地 80 会被转发到 SERVER:8080
ssh -L 8080:localhost:80 SERVER -o ExitOnForwardFailure=yes

# 访问 SERVER:80 会被转发到本地 8080 端口
# -g 允许外部访问
ssh -g -R 80:localhost:8080 SERVER -o ExitOnForwardFailure=yes
```

允许端口转发

__/etc/ssh/sshd_config__

```conf
AllowAgentForwarding yes
AllowTcpForwarding yes
# 允许 -g
GatewayPorts yes
```
