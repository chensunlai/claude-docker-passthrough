---
name: docker-vless-desktop
description: 构建或维护双容器 VLESS Reality 网关与 Ubuntu GPU 远程桌面，使桌面容器通过境外出口访问网络，并提供 SSH、VNC 和连续端口转发。适用于复用 claude-gateway / claude-passthrough 方案。
---

# 双容器代理桌面

按下面的默认要求实施；用户的新指令优先。工作文件放在 `/home2/chen/Project/` 下，不放进无关项目。

## 要求

- 只用两个容器：`claude-gateway` 负责 sing-box TUN、VLESS Reality 和入站转发；`claude-passthrough` 是完整 Ubuntu 24.04 用户空间，带 XFCE、Chrome、SSH、TigerVNC、noVNC。
- app 独立网络命名空间，只显示 `eth0` 和 `lo`。不用 `network_mode: service:proxy`，不增加 sandbox 容器。TUN 只在网关中。
- app 全部出站 TCP、UDP、DNS 经远端节点；代理失效时不得直连。宿主机和其他容器保持原网络。
- 全英文：`LANG=LC_ALL=en_US.UTF-8`、`LANGUAGE=en_US:en`，浏览器 `en-US`；清除桌面、浏览器、noVNC 的中文翻译资源。时区取出口所在地；现有部署为 `America/Los_Angeles`，不是宿主机时区。
- 用户 `chen`，密码按用户提供值设置，允许免密 sudo，禁用 root SSH 登录。VNC 密码独立设置或保留，不把任何密码、UUID、节点密钥写入 skill。
- 无宿主机目录挂载、Docker socket、host 网络/PID、privileged 或 SYS_ADMIN。允许 NVIDIA runtime 注入 GPU 设备和驱动库；`gpus: all`，驱动能力 `compute,utility`，`shm_size: 1gb`。Docker 仍共享宿主机内核。

## 网络与端口

当前部署默认值如下；新主机先确认地址不冲突，修改时同步配置、路由、防火墙和端口映射。

| 项目 | 默认值 |
|---|---|
| 宿主机服务地址 | `10.82.1.221` |
| 网关上联网卡 | `eth0`，`10.11.0.2/24`，上游 `10.11.0.1` |
| 网关内网网卡 | `eth1`，`10.24.7.1/24` |
| app 网卡与路由 | `eth0`，`10.24.7.138/24`，默认网关 `10.24.7.1` |
| SSH / VNC / noVNC | 宿主机 `2222 → 22`、`5901 → 5901`、`6080 → 6080` |
| 预留应用端口 | `62000–62031`，TCP 和 UDP 同端口转发 |

内网 bridge 设置 `internal: true` 和 `com.docker.network.bridge.gateway_mode_ipv4: isolated`，避免给宿主机桥分配内网 IP。当前 Docker IPAM 保留网关地址为 `10.24.7.254`，app 的实际默认路由必须指向代理 `.1`。网关上联设最高 `gw_priority`；用 `interface_name` 固定接口名，要求 Docker/Compose 支持这些字段。

端口全部发布在网关，通过网关容器内 nftables DNAT 到 app，向 app 转发时 SNAT 为 `10.24.7.1`，保证 SSH/VNC 回包走原路；app 看到的来访地址因此是网关。应用应监听 `0.0.0.0`。预留连续至少 16 个空闲端口，避开宿主机临时出站端口范围；冲突时换一段，不额外维护宿主机 NAT 服务。

## 最短构建流程

1. **读取现状与参数。** 现有工程在 `/home2/chen/Project/vless-tun-test/`，优先读 `compose.yaml`、`sing-box/start-proxy.sh`、`desktop/start-app-network` 和 `README.md`。检查 Docker 权限、`/dev/net/tun`、NVIDIA runtime；docker 组已添加但当前会话未刷新时用 `sg docker -c 'docker …'`。从已有私有配置或用户提供内容取得节点 IP/端口、UUID、Reality public_key、short_id、server_name；节点域名先解析，连接配置使用 IP。
2. **构建网关。** 固定可用 sing-box 版本或 digest，配置与入口脚本 COPY 进镜像，不 bind mount。TUN 使用 `address`、`auto_route`、`strict_route`，路由 `auto_detect_interface`，默认出口 VLESS，启用 Reality、uTLS chrome、`xtls-rprx-vision` 和 `xudp`。现有 sing-box 1.14.3 用 `dns_mode: disabled`，让 app 的外部 DNS 查询作为普通流量进入 TUN；切换版本时核对字段兼容性。网关需 NET_ADMIN、TUN 设备和 IPv4 forwarding。
3. **限制绕行并转发。** 网关先加载 nftables 再启动 sing-box。上联仅允许节点 IP/端口及既有连接；forward 默认拒绝，仅允许 app → TUN、已建立连接和已 DNAT 的入站连接；拒绝 app 新建连接访问网关本机。私网地址排除 TUN 路由以保证入站回包，同时由防火墙阻断 app 经上联访问宿主机/LAN。节点地址变化时同步更新配置和防火墙。app 显式使用 `1.1.1.1`，避免 Docker 内置 DNS 绕过代理。当前 app 只配置 IPv4，不宣称支持 IPv6。
4. **构建桌面。** 从官方 Ubuntu 安装 XFCE、Chrome、TigerVNC、noVNC/websockify、openssh-server、sudo、locales、tzdata、iproute2 和 util-linux；设置英文、时区、用户与服务入口。SSH、桌面由入口脚本启动，无需容器 systemd。以普通用户运行浏览器。现有 Chrome 使用 `--no-sandbox` 兼容容器，属于浏览器隔离减弱的取舍，不能通过给整个容器 privileged/SYS_ADMIN 来解决。NVIDIA runtime 注入设备和驱动库，不等于安装完整 CUDA Toolkit；应用所需 CUDA 用户态库、nvcc/PyTorch 按需安装。
5. **设置 app 启动网络。** 临时授予 NET_ADMIN，入口执行 `ip route replace default via 10.24.7.1 dev eth0` 并设置 DNS，随后用 `setpriv --bounding-set=-net_admin --inh-caps=-net_admin --ambient-caps=-net_admin` 启动服务。这样 SSH、桌面及 sudo 子进程均不能恢复 NET_ADMIN。不要启用会阻止用户 sudo 的 `no-new-privileges`。
6. **构建启动并交付。** `docker compose build` 后 `docker compose up -d --no-build`。只做与改动相关的必要检查：出口、DNS/UDP、SSH/VNC、GPU；用户明确要求跳过功能验证时照做，注明未验证，不把容器启动成功当成功能通过。交付连接地址、端口段、密码获取方法和数据保存方式。

## 更新与复用

- 当前 Compose 的 `desktop/Dockerfile.network` 基于本机私有快照；它不能在新主机直接从零构建。全新安装应将 `desktop/Dockerfile`、`desktop/Dockerfile.ssh`、`desktop/Dockerfile.network` 的必要安装和入口合并为从 `ubuntu:24.04` 开始的构建，移除历史快照依赖，直接创建 `chen`。已有部署则优先复用镜像层，避免重复安装整套桌面。
- 数据保存在 app 可写层：restart 保留，删除/重建会丢失新增文件。重建前停止 app 并 `docker commit` 到唯一私有快照标签，再让新镜像以该快照为基础；保留用户文件、VNC 密码和同一实例的 SSH 主机密钥。复制成独立新实例时重新生成主机密钥。
- 含账户、节点配置或用户文件的镜像保持私有。修改少量脚本时只叠加薄层；不为普通调整重装系统，不删除无关容器或网络。
- 当前访问方式：`ssh -p 2222 chen@10.82.1.221`；VNC `10.82.1.221:5901`；网页 `http://10.82.1.221:6080/vnc.html?language=en`。已有 VNC 密码可从 app 的 `/home/chen/.vnc/initial-password` 读取。
