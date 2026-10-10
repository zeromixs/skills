---
name: home-network
description: 当用户提到 home-network、家庭 Ubuntu 主机网络、zeromixs@100.109.165.99、Shadowsocks/ss-local、tun2socks、开机代理、DNS 污染、Steam 直连、局域网网关或家庭网络排障时使用。先检查实际运行状态，再做可回滚的网络维护；本技能记录当前拓扑、验证命令与未完成事项。
---

# home-network：家庭主机网络维护

目标机器：`zeromixs@100.109.165.99`（Tailscale）；Ubuntu 主机 LAN IP `192.168.3.88`，物理网卡 `enp3s0`，上游网关 `192.168.3.1`。远程路径 `~/mango/home-network/`；此路径是**远程机器上的项目**，不是本机同名源码目录。每次先 SSH 核实，不将下列 IP、服务状态当成永远不变的事实。

## 当前架构及边界（2026-10-10 实测）

- AWS 新加坡 EC2 Shadowsocks 服务端由 `~/mango/home-network/main.tf` 和本地 Terraform state 管理，客户端连接信息在 `client/ss-local-config.json`（**有密码，不得输出/写进技能**）。实例 IP 当前为 `13.212.104.24`，客户端用 TCP/UDP `48891`；AWS 机器上实际运行的是 `ss-server-48891.service`，**不是**默认的 `shadowsocks-libev.service`。先 `ssh -i ~/mango/home-network/id_rsa ubuntu@<IP>` 核对再操作。Terraform 的 `apply/destroy` 可能改 IP、密码/实例；必须先 plan、确认再执行。
- 家庭主机系统服务 `home-network-ss-local.service`（端口 `127.0.0.1:1080`，系统级 `User=zeromixs`，`LimitNOFILE=65535`），已取代用户服务 `ss-local.service`。现用 `ss-local-config.json` **无 ACL**；旧 `cn-whitelist.acl` 只是遗留资料，不是当前分流入口。
- `home-network-dns.service`：`client/dns-via-socks.py` 监听 `127.0.0.1:5354` UDP/TCP，借本地 SOCKS5 对可信 DNS 做 TCP 查询；dnsmasq 监听 `127.0.0.1:53`，国内域名在 `client/dnsmasq-cn-domains.conf` 指向国内 DNS，其他走 `127.0.0.1#5354`。`/etc/resolv.conf` 指向 `127.0.0.1`。备份 `/etc/home-network-dns.conf.pre-trusted-dns` **不得移回 `/etc/dnsmasq.d/`**：dnsmasq 会将其误读为第二份配置并启动失败。
- `home-network-transparent.service` 调用 `client/transparent-local.sh start|stop|status`，启动 `tun2socks-linux-amd64`；IPv4 仅本机 OUTPUT：国内 IP 集合 `home_cn_v4`（来源 `client/cn_cidrs.txt`）、本地/内网、SS 服务端与国内 DNS 走原物理路由；其余打 `0x4000` 进路由表 `201` → tun0。**main 表的物理默认路由始终是 `via 192.168.3.1 dev enp3s0`**，不要将 tun0 写入 main 默认路由。规则链 `HOME_PROXY_V4`；启动前做 SOCKS5/DNS 检查，启动后测试无显式代理 Google 204、BigModel 200；异常会清理。
- 已启用上述系统服务和 dnsmasq 的开机启动，停启验收成功；**尚未真实重启机器验收**。代理仅保证本机 IPv4：IPv6 Google 尚不通；局域网设备未接入（dnsmasq 只监听 loopback，FORWARD 仍由 Docker/Tailscale 管理且默认 DROP），**绝不能声称全家网络已代理**。
- Steam 当前与浏览器均属 `zeromixs` 用户；Steam 及 steamwebhelper 在 GNOME 的 `app-gnome-steam-*.scope` cgroup 中。用户要求 Steam 客户端+商店+下载+游戏**全部直连**，但**尚未实现/验证**；用户要求暂不关闭 Steam。不能按 `--uid-owner zeromixs` 绕过代理（会误伤浏览器），不能仅凭 Steam 域名/IP 清单声称覆盖所有游戏；DNS 也需隔离。修改 Steam 启动方式前先确认用户同意关闭/重新启动且无游戏下载或游戏进行中。

## 检查与验证（只读优先）

```bash
ssh zeromixs@100.109.165.99 'ip -4 route show default; ip -4 rule show; systemctl is-enabled home-network-ss-local home-network-dns home-network-transparent dnsmasq; systemctl is-active home-network-ss-local home-network-dns home-network-transparent dnsmasq; sudo -n bash ~/mango/home-network/client/transparent-local.sh status'
ssh zeromixs@100.109.165.99 'dig @127.0.0.1 www.google.com A +short +time=3 +tries=1 | head; dig @127.0.0.1 www.baidu.com A +short +time=3 +tries=1 | head; curl --noproxy "*" -4 -sk --max-time 8 -o /dev/null -w "google=%{http_code}\n" https://www.google.com/generate_204; curl --noproxy "*" -4 -sk --max-time 6 -o /dev/null -w "bigmodel=%{http_code}\n" https://open.bigmodel.cn; curl -sk --socks5-hostname 127.0.0.1:1080 --max-time 8 -o /dev/null -w "SOCKS-google=%{http_code}\n" https://www.google.com/generate_204'
```

期望：物理默认路由保持不变，Google `204`、BigModel `200`、SOCKS5 Google `204`。不要只凭一次 `curl` 判断稳定；真实浏览器需打开 Google 搜索页（`/generate_204` 返回空内容，Chromium 的 `page.goto` 可能报 `ERR_ABORTED`，不代表 Google 不可达）。验证国内百度、Docker、SSH/Tailscale 及 IPv6 的边界另行做。

服务日志：`sudo journalctl -u home-network-{ss-local,dns,transparent} -n 50 --no-pager`，dnsmasq 用 `sudo journalctl -u dnsmasq -n 50 --no-pager`。服务启停：`sudo systemctl stop|start home-network-transparent`；它的 `ExecStop` 调用脚本清理，正常不用手动改 iptables。服务单元位于 `/etc/systemd/system/home-network-{ss-local,dns,transparent}.service`，DNS 配置实际位于 `/etc/dnsmasq.d/home-network.conf`。旧 `client/tun2socks.sh` 有过错误的 `--uid-owner`、`RETURN` 分流和重复挂链问题，**不要再次运行**；以 `transparent-local.sh` 为准。

## 操作流程与安全边界

1. **变更前**：记录 `ip route show default`、服务状态、Google/BigModel/SOCKS5 检查；确认是否有其他会话/人同时在改路由。之前同机 Mango 会话同时重启 tun2socks，曾造成排障互相覆盖。避免 `pkill -f tun2socks`（会匹配到执行命令的 shell），用 systemd 单元停服务。
2. **改配置**：先读原文件、备份到不会被服务自动加载的位置（例如 `/etc/` 而非 `/etc/dnsmasq.d/`），先做 `bash -n` / `python3 -m py_compile` / `dnsmasq --test -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new` 和 `systemd-analyze verify`。只修改对应服务；不要覆盖 Docker、Tailscale、wg0 的路由/NAT/iptables。
3. **改路由**：小范围定向探测（仅一个目的 IP 的 tun 路由）→ 单 IP 的 OUTPUT 打标+表 201 → 真实目标验证 → 才考虑扩展；任何失败先恢复默认物理路由与国内 DNS，再诊断根因。不要反复叠加 `iptables -A OUTPUT` / `ip rule add`。
4. **验收**：停/启服务后规则恰好一份，物理默认路由不变，Google、BigModel、国内网站、SOCKS5、真实浏览器正常。重启验收**需用户明确允许**；未经实测不能宣称重启后已完全恢复。局域网网关/DHCP 需先选试点设备 IP 并单独确认，不擅自修改路由器。
5. **故障恢复**：`sudo systemctl stop home-network-transparent` → 验证物理默认路由及 BigModel 200；若 DNS 异常，检查 `home-network-ss-local`、`home-network-dns`、`dnsmasq`。域名不通先用 `dig @127.0.0.1` 定位，不直接重启整个网络栈。系统服务在登录前运行，不依赖用户级 `ss-local.service` 的 linger。

## 已知坑与待办

- 原本 dnsmasq 国内上游解析 Google 会得到被污染的地址；现其非国内域名通过 SOCKS5 查询可信 DNS。`curl --socks5-hostname` 能访问 Google **不证明**本机 DNS 或 TUN 正常，需分别查。
- `ip route get <IP>` 不带 `mark` 返回 main 路由，**不能证明** mangle OUTPUT 打标无效；需配合 `ip route get <IP> mark 0x4000`、iptables 计数器和实测流量。SS 服务端 ICMP ping 超时也**不能说明** TCP 或服务不可达；应看 SSH、服务端监听、SOCKS5 实测。
- 主机有 IPv6 默认路由，当前仅 IPv4 代理；Google IPv6 直连失败。Steam 全直连、局域网试点、真实重启验证以及运行中 tun2socks 单独退出时的恢复机制尚未完成。
- DNS 经可信上游失败时国内白名单可能仍可解析，其他域名会失败；代理服务是否自恢复要看日志/健康检查，不能只看 `active`。
- Terraform SSH 私钥、SS 密码、Mango token 都是敏感资料：不在对话、日志和技能中回显。曾在排障输出中泄露旧凭据，后续安排轮换，但轮换需与客户端、服务端同步并经用户确认。
- 移动蜂窝网络（223.104.x 段出口）到境外 IP 高端口的 TCP SYN-ACK 回包被运营商丢弃（表现为「只有上传没下载」/timeout），WiFi（120.244.x）正常。缓解：AWS 服务器已手工加开 **443 端口** ss 实例（`shadowsocks-libev-port443`，terraform 之外配置，**重建实例后需重做**：/etc/shadowsocks-libev/config-port443.json + systemd 单元 + 安全组放行 443/tcp+udp）。移动端（V2box 等）蜂窝网络下用 443，桌面端继续 48891。诊断方法：服务端 `tcpdump -i ens5 -nn port <端口>` 看 SYN-ACK 是否重传无 ACK。
