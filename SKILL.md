---
name: docker-vless-desktop
description: 在任意兼容的 Linux Docker 服务器上构建或维护双容器 VLESS Reality 网关与 Ubuntu GPU 远程桌面，提供独立代理出口、SSH、VNC 和连续端口转发。
---

# 双容器代理桌面

生成可在目标服务器从零构建的 Docker Compose 工程。按用户指定目录或当前合适目录存放工程；不要求固定路径，不修改 Docker 数据目录。所有部署参数从目标环境和用户输入取得，不依赖原服务器的文件、镜像、账户或地址。

## 要求

- 只用两个服务：`gateway` 负责 sing-box TUN、VLESS Reality 和入站转发；`app` 是完整 Ubuntu 用户空间，带 XFCE、浏览器、SSH、TigerVNC、noVNC。默认让 Compose 管理容器名，用户指定时再设置。
- app 独立网络命名空间，只显示 `eth0` 和 `lo`；TUN 仅在 gateway 中。不共享 gateway 网络命名空间，不增加 sandbox 容器。
- app 全部出站 TCP、UDP、DNS 经远端节点；代理失效时不得直连。宿主机和其他容器保持原网络。
- 全英文：`LANG=LC_ALL=en_US.UTF-8`、`LANGUAGE=en_US:en`，浏览器 `en-US`，清除桌面、浏览器和 noVNC 的中文翻译资源。`TZ` 使用出口所在地的 IANA 时区，由用户指定或根据出口位置确定。
- 普通用户名、SSH 密码或公钥由用户提供；允许免密 sudo，禁用 root SSH 登录。VNC 单独设置密码。凭据不写入 skill 或公开仓库。
- 默认无宿主机目录挂载、Docker socket、host 网络/PID、privileged 或 SYS_ADMIN。GPU 主机通过 NVIDIA Container Toolkit 提供设备与驱动库，启用 `gpus: all`、驱动能力 `compute,utility` 和 `shm_size: 1gb`。无 GPU 时说明限制；用户同意后可按 CPU 模式部署。

## 部署参数

集中管理以下参数，贯穿 Compose、配置和入口脚本，避免多处硬编码：

- 节点 IP、端口、UUID、Reality public_key、short_id、server_name；域名节点先解析为 IP。
- `APP_USER`、登录凭据、`TZ`、外部 DNS 地址。
- 不与目标主机路由及已有 Docker 网络重叠的上联/内网子网、网关内网 IP、app IP、Docker IPAM 保留网关地址。
- 宿主机监听地址、SSH/VNC/noVNC 发布端口、连续至少 16 个应用端口。检查实际占用及宿主机临时出站端口范围，冲突时换一段。

监听地址取目标服务器的可达地址，或按用户要求监听所有地址。容器 SSH/VNC/noVNC 默认端口为 `22/5901/6080`；宿主机端口按可用情况选择。网关/app 地址和时区均不固定为某次部署的值。

## 最短构建流程

1. **准备。** 检查 Docker Engine、Compose、`/dev/net/tun`、CPU 架构与 GPU runtime。确认 Engine/Compose 支持 `interface_name`、`gw_priority`、bridge isolated 模式及所用 GPU 配置。工程只需 `compose.yaml`、两个 Dockerfile、sing-box 配置和服务入口脚本；非敏感参数可放 `.env`，凭据独立保存并排除版本控制。
2. **构建网关。** 使用固定版本或 digest 的官方 sing-box 镜像，COPY 配置和入口脚本，不 bind mount。TUN 设置 `address`、`auto_route`、`strict_route`，路由启用 `auto_detect_interface`，默认出口 VLESS Reality，使用 uTLS chrome、`xtls-rprx-vision` 和 `xudp`。字段必须匹配所选版本；支持 `dns_mode: disabled` 的版本可关闭 TUN 自动 DNS 劫持，让 DNS 作为普通代理流量处理。网关授予 NET_ADMIN、TUN 设备和 IPv4 forwarding。
3. **连接网络与转发。** gateway 接上联 bridge（`eth0`）与内网 bridge（`eth1`），上联具有最高 `gw_priority`；app 只接内网（`eth0`）。内网设置 `internal: true` 和 `com.docker.network.bridge.gateway_mode_ipv4: isolated`，不为宿主机桥分配内网 IP。Docker IPAM 保留网关地址与 gateway 容器 IP 分开，app 实际默认路由指向 gateway 容器。全部端口通过普通 Compose `ports` 发布在 gateway，再用其容器内 nftables DNAT 到 app；向 app 转发时 SNAT 为 gateway 内网 IP，保证回包返回。预留端口段同时转发 TCP/UDP，不另外维护宿主机 NAT 服务。
4. **阻止绕行。** gateway 先加载防火墙再启动 sing-box。上联只允许节点 IP/端口与既有连接；forward 默认拒绝，仅允许 app → TUN、已建立连接和已 DNAT 的入站连接；拒绝 app 新建连接访问网关本机。私网路由排除 TUN 以保证入站回包，同时阻断 app 经上联访问宿主机/LAN。节点 IP 更新时同步修改防火墙。app 直接使用外部 DNS，避免 Docker 内置 DNS 绕过代理。默认仅配置 IPv4；需要 IPv6 时必须同时实现代理路由与防泄漏规则。
5. **构建桌面。** 从官方 Ubuntu LTS（如 `ubuntu:24.04`）直接安装 XFCE、适配目标架构的浏览器、TigerVNC、noVNC/websockify、openssh-server、sudo、locales、tzdata、iproute2 和 util-linux。设置用户、英文、时区和服务入口，无需 systemd。浏览器以普通用户运行；仅在确有兼容性问题时使用 `--no-sandbox` 并说明取舍，不为此扩大容器权限。NVIDIA runtime 不等于完整 CUDA Toolkit，应用所需 CUDA 用户态库、nvcc/PyTorch 按需安装。新实例首次启动生成 SSH 主机密钥和 VNC 凭据，同一实例重启时保留。
6. **启动。** app 临时授予 NET_ADMIN，入口设置 `ip route replace default via "$GATEWAY_IP" dev eth0` 和外部 DNS，再以 `setpriv --bounding-set=-net_admin --inh-caps=-net_admin --ambient-caps=-net_admin` 启动 SSH 和桌面服务，避免 sudo 子进程恢复 NET_ADMIN。不启用会阻止 sudo 的 `no-new-privileges`。配置网关就绪后启动 app，使用 `restart: unless-stopped`。执行 `docker compose up -d --build`。

## 交付与维护

- 只做必要检查：app 仅有 eth0/lo、TCP/UDP/DNS 出口、SSH/VNC、GPU。用户要求跳过时注明未验证，不把启动成功当成功能通过。
- 按实际参数交付 SSH 命令、VNC 地址、noVNC URL、端口段和凭据获取方法。应用端口需监听 `0.0.0.0`；经 DNAT/SNAT 的入站连接在 app 中显示为网关来访。
- 默认数据保存在容器可写层：restart 保留，删除或重建会丢失新增文件。更新前停止 app 并快照或备份，再恢复到新容器；用户需要长期持久化时可选择 Docker 命名卷，按其隔离要求实施。
- 全新部署必须从公开基础镜像构建，不依赖历史快照或本机专用文件。私有快照只用于已有实例的数据保留；迁移时可导出/导入，或重新构建后恢复备份。独立克隆实例应重新生成 SSH 主机密钥。
- 含节点配置、账户或用户文件的镜像保持私有。复用构建缓存，不重装无关组件，不修改无关容器和网络。Docker 仍共享宿主机内核。
