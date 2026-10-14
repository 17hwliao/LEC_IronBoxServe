# 连接与维护手册

基线日期：2026-10-14。公网入口和凭据由管理员私发，不进入公开仓库。

含 PUBLIC_SSH_HOST / PUBLIC_SSH_PORT 的命令是模板，**先替换占位符**。注意区分 Windows 电脑、PVE 宿主机和 CT 103 内部，Linux 服务命令不能直接运行在 Windows PowerShell。

## SSH 与 SFTP

Windows PowerShell / SSH 客户端，局域网普通账号：

```bash
ssh lec@192.168.5.7
```

公网普通账号：

```bash
ssh -p PUBLIC_SSH_PORT lec@PUBLIC_SSH_HOST
```

管理员换为 root。强制只测试密码认证：

```bash
ssh -p PUBLIC_SSH_PORT -o PubkeyAuthentication=no -o PreferredAuthentications=password root@PUBLIC_SSH_HOST
```

作用：通过公网隧道实际验证密码，不把密钥登录误认为密码验证。密码在交互提示处输入，不附在命令行。

HexHub 等客户端填写 SSH、管理员私发的主机 / 端口、lec、密码认证；SFTP 参数相同。多人可同时使用 lec，但同一 UID 不隔离文件、进程或操作身份。不要向普通协作者发 root 或樱花管理凭据。

首次连接核对 SSH 主机指纹；指纹变化不能直接删除 known_hosts 跳过校验。通过隧道上传下载计入所属樱花账号额度；LAN 直接访问 .7:22 不经过该管理隧道。

## 远程访问 PVE 网页

不另公开 8006。在 **Windows PowerShell** 使用管理员账号：

```bash
ssh -N -L 127.0.0.1:18006:127.0.0.1:8006 -p PUBLIC_SSH_PORT root@PUBLIC_SSH_HOST
```

作用：只在当前电脑回环监听 18006，经加密 SSH 转发到 PVE 网页。输入密码后无命令提示符是正常的，保持窗口打开。

浏览器访问：

```text
https://127.0.0.1:18006/
```

PVE 用户名 root、领域 Linux PAM、宿主机 root 密码。lec 当前没有配置 PVE 网页权限。自签名 / 地址不匹配可产生证书警告；仅在 SSH 身份和 URL 已确认时继续。

关闭窗口仅停止本电脑的网页转发，不停止服务器或业务。LAN 直接打开 https://192.168.5.7:8006/。

## 宿主机穿透服务

以下在 **PVE 宿主机 Shell** 运行，只读检查：

```bash
systemctl status natfrp.service --no-pager
systemctl list-timers natfrp-ssh-watchdog.timer --no-pager
journalctl -u natfrp.service --since "-10 minutes" --no-pager -n 50
```

分别查看启动器、定时器和近期日志。原始日志可能包含来源 IP 和私密信息，分享前脱敏。

仅在维护已获准并知道会短暂断开公网连接时：

```bash
systemctl restart natfrp.service
```

作用：重启宿主机管理穿透并重新加载配置，不是重启物理服务器。最好保留 LAN 恢复入口。

真正停机维护时，先阻止 watchdog 拉起：

```bash
systemctl disable --now natfrp-ssh-watchdog.timer
systemctl stop natfrp-ssh-watchdog.service
systemctl stop natfrp.service
```

恢复：

```bash
systemctl start natfrp.service
systemctl enable --now natfrp-ssh-watchdog.timer
```

不能只靠即将被关闭的公网连接停服务而没有其他恢复手段。关闭官网页面不停止穿透；官网“停止隧道”是实际操作。watchdog 只看子进程存在，不能证明节点、授权和实际登录正常。

## 连接错误如何定位

记录失败时间、入口、账号和完整错误，再在 **宿主机** 只读查询：

```bash
journalctl -u ssh.service --since "-10 minutes" --no-pager -n 50
journalctl -u natfrp.service --since "-10 minutes" --no-pager -n 50
```

- Connection refused：端口没有接受连接或被拒绝，不等于密码错误。
- EOF / Connection closed：握手中断，结合两侧日志判断，不仅凭报错认定服务崩溃。
- Failed password：请求到了 sshd，才核对对应账号及有效认证配置。
- SSH 成功、网页失败：核对本地转发和 pveproxy；SSH 端口不是 HTTPS 端口。

穿透连接到宿主机回环，sshd 可能显示 127.0.0.1，不能仅凭它识别具体访问者。

如重新启用樱花访问密码，它不是 Linux 密码、账户访问密钥或远程管理密码；每个来源 IP 需授权，frpc 重启会清缓存。[官方认证说明](https://doc.natfrp.com/bestpractice/frpc-auth.html)

## 用户权限检查

在 **宿主机**：

```bash
id lec
groups lec
sudo -l -U lec
```

确认用户、组和 sudo 权限。直接运行 groups 只显示当前登录用户的组，不是全部用户。

查看 root 的回环匹配有效配置：

```bash
/usr/sbin/sshd -T -C user=root,host=pve,addr=127.0.0.1 | grep -E '^(permitrootlogin|passwordauthentication|authenticationmethods|allowtcpforwarding) '
```

若 Match 条件改变需代入实际来源。不要把 lec 加入 sudo，也不把“普通读写权限”误写成命令白名单或人员隔离。

## 业务 CT 103 的只读检查

在 **宿主机**：

```bash
pct status 103
pct config 103
pct exec 103 -- ip -4 -br addr
pct exec 103 -- docker ps
pct exec 103 -- systemctl status frpc.service --no-pager
```

查看 CT 状态、配置、租约地址、Docker 和业务 FRP。宿主机 natfrp 与 CT 的 frpc 是独立服务，不混淆。

不公开业务 .env、数据库 / RustFS 数据或密钥，不运行会打印完整 Docker environment 的 inspect 来分享日志。

Docker unless-stopped 策略以 CT 先启动为前提；宿主机 pve-guests 当前 masked。维护重启前需制定 CT 启动恢复安排，不能假定 onboot: 1 足够。本页不附自动启用或重启生产业务的命令。

## 版本、存储与变更边界

在 **宿主机** 只读检查：

```bash
pveversion
uname -r
cat /etc/debian_version
dpkg --audit
df -h / /mnt/pve/Data /mnt/pve/System
pvesm status
systemctl --failed --no-pager
```

无 dpkg audit 输出不证明版本受支持；存储 active 不证明独立挂载或已备份。PVE 主仓库当前缺失，不能从 apt“没有更新”推断完整更新。

暂不升级生产宿主机。修源、升级、清理包 / 磁盘、调整防火墙、启用客户机自启动和重启均须另行授权。

变更前明确协作者是否在线、业务停机窗口、外部备份、LAN / 现场 / IPMI 恢复入口及 CT 103 启动方式。备份覆盖 PostgreSQL、RustFS、秘密部署配置和必要宿主机配置，并验证恢复；GitHub 文档或本机备份目录不是外部可恢复备份。
