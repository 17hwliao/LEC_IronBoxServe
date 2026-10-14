# 当前配置快照

核对日期：2026-10-14 上午（Asia/Shanghai）。本页是脱敏运行摘要，不是配置文件全量转储。占位符不能直接用于生产。

## 1. 宿主机与硬件

| 项目 | 实测结果 |
| --- | --- |
| 主机 | pve；独立单节点，无集群 |
| 系统 | Debian GNU/Linux 12 / Bookworm；点版本 12.15 |
| PVE 包 | pve-manager 8.4.21；proxmox-ve 8.4.0；qemu-server 8.4.10；pve-container 5.3.6 |
| 内核 / 架构 | 6.8.12-43-pve；x86_64 |
| CPU | 两颗 Xeon E5-2670 v2 @ 2.50 GHz；每颗 10 核 / 20 线程 |
| 内存 / swap | 内存 67,385,999,360 bytes，约 62.76 GiB；宿主机 swap 8 GiB |
| 虚拟化 | CPU 支持 VT-x |
| 网卡 | 双端口 Intel I350 Gigabit：enp7s0f0、enp7s0f1 |
| 存储控制器 | Broadcom / LSI SAS2308；另有 Intel C600/X79 SATA RAID 控制器 |
| 显示 | ASPEED Graphics Family |
| 型号 | 现场记录为浪潮旧服务器；DMI 厂商 / 型号均为“.”，未确认具体型号 |

物理硬盘数量、RAID 级别、硬盘健康和 IPMI 登录 / 电源控制尚未验证。CPU 功能表有 AVX，未列出 AVX2；要求更新指令集的软件不能仅凭 Debian 版本承诺兼容。

## 2. 磁盘、卷组与 PVE 存储

设备名只代表本次枚举，后续可能变化；禁止据此直接格式化或清理设备。

| 本次设备 / 卷 | 内容 | 当前用途 |
| --- | --- | --- |
| sda，约 1.8 TiB，显示 Logical Volume | 1 GiB XFS 分区；centos VG 的 root 50 GiB、swap 31.5 GiB、home 约 1.74 TiB | 旧 CentOS 保留，未挂载为当前系统 |
| sdb，约 1.8 TiB，显示 MG03SCA200 | BIOS boot 小分区、512 MiB FAT 分区及 pve-OLD-AFD96CDA VG | 当前 PVE 和客户机磁盘 |
| pve-OLD-AFD96CDA/root | 96 GiB ext4 | 挂载 /；df 可用约 74 GiB |
| pve-OLD-AFD96CDA/swap | 8 GiB | 当前 swap |
| pve-OLD-AFD96CDA/data | 约 1.67 TiB thin pool | Data% 约 6.12%，Meta% 约 0.37% |

根设备为 `/dev/mapper/pve--OLD--AFD96CDA-root`，fstab 按根卷和 swap 的 UUID 定位。保留现有 VG 名，不擅自改回 pve。

`/etc/pve/storage.cfg` 的非秘密配置：

```text
dir: local
    path /var/lib/vz
    content backup,vztmpl,iso

lvmthin: local-lvm
    thinpool data
    vgname pve-OLD-AFD96CDA
    content images,rootdir

dir: Data
    path /mnt/pve/Data
    content images
    prune-backups keep-all=1
    shared 0

dir: System
    path /mnt/pve/System
    content vztmpl,iso,snippets,rootdir,images,backup
    prune-backups keep-all=1
    shared 0
```

四个存储 ID 都 active。但 findmnt 确认 Data、System 没有独立挂载，连同 local 都占用根文件系统，**容量不能按存储 ID 累加**。本地备份不等于独立故障域备份。

旧 VM 100、101、102、103、106 的部分 thin volumes 及 101 的快照仍可见；没有完成所有旧数据清理。特别是 `vm-103-disk-1` 已作为当前业务 CT 103 根盘，不能误删。

## 3. 宿主机网络与端口

| 项目 | 当前值 |
| --- | --- |
| 接入网口 | enp7s0f0 UP；enp7s0f1 DOWN |
| 桥 / 地址 | vmbr0 → enp7s0f0；192.168.5.7/24 静态 IPv4 |
| 默认网关 | 192.168.5.1，经 vmbr0 |
| 主机映射 | /etc/hosts：192.168.5.7 pve.lan pve |
| DNS / 搜索域 | 114.114.114.114 / lan |

`/etc/network/interfaces` 有效配置：

```text
auto lo
iface lo inet loopback

iface enp7s0f0 inet manual
iface enp7s0f1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.5.7/24
    gateway 192.168.5.1
    bridge-ports enp7s0f0
    bridge-stp off
    bridge-fd 0
```

路由器是否做了 DHCP 排除 / 地址保留未确认。vmbr1 未配置，旧 VM 如引用它不能直接假定可启动。

| 监听地址 / 端口 | 服务 | 说明 |
| --- | --- | --- |
| 0.0.0.0:22、[::]:22 | SSH / SFTP | root 和 lec 远程接入 |
| *:8006 | PVE HTTPS | 本机 HTTP 200；公网经 SSH 转发 |
| 127.0.0.1:7102 | SakuraFrp WebUI | 不对公网另建穿透 |
| 127.0.0.1:85 | pvedaemon 内部 API | 内部监听 |
| *:3128 | PVE SPICE proxy | 不表示存在公网映射 |
| TCP / UDP 111 | rpcbind | 监听主机接口 |
| 回环 25、UDP 323 | 本地邮件、chrony | 回环监听 |

UFW 单元 active/enabled，但 ufw status 是 inactive；PVE firewall 单元 active/enabled，但状态是 disabled/running。iptables 三个基础链策略 ACCEPT，nft ruleset 为空。**单元 active 不等于已开启防火墙过滤。** 未确认路由器全部映射，也未在本次调整规则。

## 4. 账号与 SSH 权限

| 账号 | 身份 | 当前权限 |
| --- | --- | --- |
| root | UID/GID 0 | 宿主机管理员，允许 SSH 密码登录，保留独立恢复公钥入口 |
| lec | UID/GID 1000，同名主组，/home/lec，/bin/bash | 只属于 lec 组，无 sudo |
| natfrp | UID/GID 997，/home/natfrp，/sbin/nologin | 宿主机启动器服务账号，不用于协作登录 |

`/home/lec` 为 0700、lec:lec；root / lec 密码状态均为 P（已设密码），不导出密码及 shadow 哈希。sudo 检查确认 lec 不获准执行 sudo。

有效设置以 `sshd -T -C` 检查为准：

- root：PermitRootLogin yes，PasswordAuthentication yes，AuthenticationMethods any。
- lec：PasswordAuthentication yes，AuthenticationMethods password。
- 两者：AllowTcpForwarding yes，DisableForwarding no，PermitOpen any；ForceCommand / ChrootDirectory 都是 none。
- root AuthorizedKeysFile 使用独立恢复公钥文件；密钥内容和本地恢复存档不进仓库。

片段 `00-lec-lan-password.conf` 和旧 `00-lec-recovery.conf` 同时存在。旧恢复片段有 prohibit-password 行，但不能只看其中一行判断最终设置。名称中的 lan 不表示限制来源 IP；当前两种账号都可通过管理隧道到达 SSH。

lec 是单个共享 Linux 用户，不是能自行创建账号的管理员，也不是 SFTP-only / chroot 用户。有普通 shell，只能按普通用户权限读写获准路径；多人使用相同 UID，不隔离文件和进程，不能承诺无法终止同账号连接。

PVE 用户记录仅有 `root@pam`，无额外用户 ACL；lec 没有配置网页管理权限。

## 5. 宿主机 systemd 服务

| 单元 | 当次运行状态 | 开机状态 |
| --- | --- | --- |
| ssh、pve-cluster、pvedaemon、pveproxy、pvestatd、pvescheduler | active | enabled |
| natfrp.service | active | enabled |
| natfrp-ssh-watchdog.timer | active | enabled |
| pve-guests.service | inactive | masked |
| cron.service | inactive | masked |
| pve-ha-crm.service、pve-ha-lrm.service | inactive | masked |
| ufw、pve-firewall | active，过滤未启用 | enabled |

systemctl --failed 无 failed 单元；不代表所有需要的服务已启用。单节点没有据此自动启用 HA 服务。

### 宿主机管理穿透

- 安装日志版本：启动器 3.1.8，frpc 0.51.0-sakura-15。
- 二进制在 /home/natfrp；配置 /etc/natfrp/config.json；用户 natfrp。
- 单个自动启动隧道 Pve，浙江电信 PLUS6，TCP → 127.0.0.1:22。
- 公网入口、隧道 ID、访问密钥及远程管理密码不公开。
- 远程管理启用；WebUI localhost:7102；frpc 日志 trace，原始日志不提交。
- 用户已关闭额外访问密码；2026-10-14 公网 root 密码登录再次通过，SSH 主机密钥匹配。没有替每个外部网络逐一验收。
- 官网是控制面板，关闭浏览器不停止穿透；流量计入隧道所属樱花账号。[远程管理](https://doc.natfrp.com/launcher/remote-v2.html)、[流量计量](https://doc.natfrp.com/faq/network.html)

实际主单元：

```ini
[Unit]
Description=SakuraFrp Launcher
After=network.target

[Service]
User=natfrp
Type=simple
TimeoutStopSec=20
Restart=always
RestartSec=5s
Environment="NATFRP_SERVICE_WD=/etc/natfrp"
ExecStart=/home/natfrp/natfrp-service --daemon

[Install]
WantedBy=multi-user.target
```

### SSH 隧道进程守护

- 脚本 /usr/local/sbin/natfrp-ssh-watchdog；单元 /etc/systemd/system/natfrp-ssh-watchdog.service。
- Type=oneshot，root 执行，TimeoutStartSec=45s。
- timer：OnBootSec=90s、OnUnitInactiveSec=60s、AccuracySec=5s，约每分钟检查一次。
- 检查 natfrp 的 MainPID 及该进程下 natfrp 用户的 frpc 子进程；缺失时重启 natfrp.service。
- timer 当次 active，存在下一次计划和前次触发记录。历史语法 / 单元校验及健康分支通过，但没有人为杀进程验证恢复。

只检查进程存在，不检查节点、授权、握手或登录；针对当前单隧道，不保证多隧道分别健康。手动停唯一隧道也可能被自动拉起，维护停机先停 timer。它不负责主机开机、业务 CT 启动、网络恢复或备份。

## 6. 业务容器与历史 VM

### CT 103：lec-sso

| 项目 | 当前配置 |
| --- | --- |
| 类型 | 非特权 LXC，amd64；Debian 12.12，共享宿主机内核 |
| 资源 | 6 核配额、8192 MiB 内存、1024 MiB swap |
| 根盘 | local-lvm:vm-103-disk-1，64 GiB |
| 特性 | nesting=1,keyctl=1 |
| 网络 | eth0 → vmbr0，IPv4 DHCP；当前 192.168.5.29/24，网关 / DNS 192.168.5.1 |
| 启动配置 | onboot: 1，startup: order=30 |
| 实际限制 | 宿主机 pve-guests masked，不能保证开机自动启动 |

地址 .29 是当前 DHCP 租约，不当作固定地址。容器内 Docker / containerd 运行，三项 Docker 业务均显示 healthy：

| 服务 | 镜像显示 | 端口 | 持久化 / 重启策略 |
| --- | --- | --- | --- |
| Logto | lec-sso-logto:production | CT 回环 3001、3002 | 未发现挂载；unless-stopped |
| PostgreSQL | postgres:17-alpine | Docker 内部 5432，未发布到 CT 接口 | named volume lec-sso-production_lec-sso-postgres → /var/lib/postgresql/data；unless-stopped |
| RustFS | 镜像 ID 7465b3199315，来源 tag 未核实 | CT 回环 9000、9001 | named volume lec-sso-production_lec-sso-rustfs → /data；unless-stopped |

业务 Docker volumes 位于 CT 内，不是宿主机 /var/lib/docker。没有公开 .env、数据库 / 对象存储凭据、业务数据或应用原始日志。healthy 只是当次运行状态，不等于 SSO 登录、上传下载等端到端验收。

CT 另有 `frpc.service`：active/enabled，用户 frp，Restart=on-failure，/etc/systemd/system/frpc.service。它与宿主机 natfrp 是独立的业务穿透；本次未导出其秘密配置或核对全部业务公网域名。

### 保留的历史 QEMU VM

| VMID | 名称 | 配置内存 | 配置主盘 | 状态 |
| --- | --- | --- | --- | --- |
| 100 | VM 100 | 2048 MiB | 4 GiB | stopped |
| 101 | VM 101 | 8192 MiB | 150 GiB | stopped |
| 102 | 20.4 | 2048 MiB | 50 GiB | stopped |
| 105 | VM 105 | 2048 MiB | 32 GiB | stopped |
| 106 | MinecraftServer | 8192 MiB | 150 GiB | stopped |

数值来自 qm list，不证明磁盘完整或系统可启动；本次不启动、清理、删除或恢复这些 VM。

## 7. 当前软件源与更新边界

有效 Debian 源：

```text
deb https://mirror.nju.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirror.nju.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirror.nju.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

源码源已注释；另有 ceph.list：

```text
deb http://download.proxmox.com/debian/ceph-squid bookworm no-subscription
```

**当前缺少启用的 PVE 主仓库。** pve-enterprise.list.bak、pve-enterprise.list.dpkg-dist 不是启用源。apt-cache policy 对 PVE 组件仅有已安装 dpkg 状态条目，不是远端仓库候选。

dpkg --audit / apt-mark showhold 无输出。已安装 tmux 3.3a-3、jq 1.6-2.1+deb12u2。本次不执行 apt update、升级、修源或 autoremove；生产修源及维护须另行授权。不能因为版本风险而自行把 bookworm 改成 trixie。

## 8. 启动与备份范围

- 本次是 Legacy / BIOS 启动，不是 UEFI。
- PVE GRUB：GRUB_DEFAULT=0，GRUB_TIMEOUT=5；内核参数 console=tty0 loglevel=3 noresume。
- 首个 Linux 启动项加载 /boot/vmlinuz-6.8.12-43-pve，root 指向现有 pve-OLD-AFD96CDA 根卷。
- 对应 initrd 存在；旧 5.15.158-2-pve、5.15.30-2-pve 内核 / initrd 保留，不自动清理。
- 2026-10-13 已验证无人值守正常进入 PVE；完整 BIOS 设备优先级、IPMI 和全部备用启动路径未在本次重测。
- 本地存在加密的配置 / 访问恢复资料，不公开路径、内容或凭据；不是生产 PostgreSQL、RustFS 和部署配置的完整外部备份。

本次只读核对不改变用户“暂不升级生产服务器”的决定。
