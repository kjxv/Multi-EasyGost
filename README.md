# Multi-EasyGost 一键脚本使用指南

这是一个用于在 Linux VPS 上安装和管理 GOST 的交互式脚本，可管理多条 TCP/UDP 转发规则，并可一键创建 Shadowsocks、SOCKS5 和 HTTP 代理。

本仓库是基于 [KANIKIG/Multi-EasyGost](https://github.com/KANIKIG/Multi-EasyGost) 的维护版本，保留原项目来源、贡献说明和 GPL-3.0 许可。当前脚本固定使用 **GOST v2.11.2**，不会自动跟随 GOST 的后续大版本升级。

## 一键部署

支持使用 systemd 的常见 Debian、Ubuntu、CentOS 等 Linux 发行版。建议先登录 VPS 并切换到 root：

```bash
sudo -i
```

然后执行：

```bash
wget -O /root/gost.sh https://raw.githubusercontent.com/kjxv/Multi-EasyGost/v2/gost.sh && chmod +x /root/gost.sh && /root/gost.sh
```

如果系统没有 `sudo`，请直接使用 root 账号登录。脚本会以 root 权限安装程序、写入 `/etc/gost` 并管理 systemd 服务。

### 再次进入菜单

```bash
/root/gost.sh
```

如果你将脚本下载到了其他目录，请改用该目录中的实际路径，例如 `./gost.sh`。

## 主菜单说明

| 选项 | 功能 | 说明 |
| --- | --- | --- |
| 1 | 安装 GOST | 安装固定版本 GOST v2.11.2、systemd 服务和默认配置 |
| 2 | 更新 GOST | 重新安装当前固定的 GOST 版本，并保留已有 `/etc/gost` 配置 |
| 3 | 卸载 GOST | 删除 GOST、配置目录和当前目录中的脚本，请谨慎使用 |
| 4 | 启动 GOST | 启动 systemd 服务 |
| 5 | 停止 GOST | 停止 systemd 服务 |
| 6 | 重启 GOST | 按原始配置重建 `config.json` 并重启服务 |
| 7 | 新增配置 | 新增转发、隧道、代理、均衡负载或 CDN 转发规则 |
| 8 | 查看配置 | 列出当前所有规则及其端口 |
| 9 | 删除配置 | 按编号删除一条规则，然后自动重建配置并重启 |
| 10 | 定时重启 | 创建或删除 GOST 定时重启任务 |
| 11 | TLS 证书 | 申请或配置用于 TLS/WSS 的自定义证书 |

## 从全新 VPS 创建一个 SOCKS5 代理

以下流程适用于一台刚开通、已获得公网 IP 的 VPS。

1. 登录 VPS，执行 `sudo -i` 切换到 root。
2. 执行上方的一键部署命令，进入脚本菜单。
3. 输入 `1`，选择“安装 GOST”。
4. 脚本询问是否使用大陆镜像时：境外 VPS 建议输入 `n`，从 GOST 官方 GitHub Release 下载；中国大陆网络访问 GitHub 困难时可输入 `y` 使用第三方二进制镜像。
5. 安装完成后，执行 `/root/gost.sh` 再次进入菜单。
6. 输入 `7`，选择“新增 gost 转发配置”。
7. 在功能列表输入 `4`，选择“一键安装 ss/socks5/http 代理”。
8. 在代理类型列表输入 `2`，选择 `socks5`。
9. 按提示依次输入：
   - SOCKS5 密码；
   - SOCKS5 用户名；
   - SOCKS5 服务端口，例如 `20001`。
10. 脚本写入配置并自动重启 GOST。客户端连接信息为：
    - 服务器：VPS 的公网 IP；
    - 端口：刚才输入的端口；
    - 用户名：刚才输入的用户名；
    - 密码：刚才输入的密码。
11. 在 VPS 服务商的安全组/云防火墙和系统防火墙中放行该端口。普通 SOCKS5 网页代理至少需要放行 TCP；使用 UDP 转发时还要放行相同的 UDP 端口。

密码和用户名会写入 `/etc/gost/rawconf` 和生成后的 `/etc/gost/config.json`，本维护版会将这两个文件权限设为仅 root 可读写。仍请使用强密码并妥善保护 VPS 的 root 权限。

## 查看、删除和增加更多配置

### 查看现有配置

重新运行 `/root/gost.sh`，输入 `8`。列表左侧的序号就是删除配置时需要使用的编号。

### 删除一条配置

重新运行脚本，输入 `9`，确认列表后输入要删除的配置编号。脚本会删除该行、重建 `/etc/gost/config.json` 并重启 GOST。

重要操作前可先备份原始规则：

```bash
cp /etc/gost/rawconf /etc/gost/rawconf.backup
```

### 创建第二个或第三个 SOCKS5

每增加一个代理，都重新执行一次“`7` → `4` → `2`”流程，并为每条代理使用不同端口，例如 `20001`、`20002`、`20003`。用户名和密码可以不同。

同一台 VPS 上创建多个端口只会得到多个代理入口，**这些代理仍共享这台 VPS 的同一个公网出口 IP**。如果需要多个不同的出口 IP，通常需要多台 VPS 或一台确实绑定了多个可用公网 IP 的服务器，并进行额外的出站绑定配置。

## 验证端口和代理出口

先在 VPS 上确认服务正在运行：

```bash
systemctl status gost --no-pager
```

查看 GOST 是否监听了预期端口，将示例端口替换为你的实际端口：

```bash
ss -lntup | grep -E ':(20001|20002|20003)\b'
```

然后在另一台电脑上测试 SOCKS5。这样既能验证公网端口是否可达，也能验证账号密码和出口 IP：

```bash
curl --proxy socks5h://VPS公网IP:20001 --proxy-user '用户名:密码' https://api.ipify.org
```

返回结果应为该 VPS 的公网出口 IP。建议再直接运行一次 `curl https://api.ipify.org`，对比“本机直连出口”和“代理出口”是否不同。Windows PowerShell 如果把 `curl` 映射成了其他命令，可将上面的 `curl` 改为 `curl.exe`。

如果测试失败，请依次检查：

1. `systemctl status gost` 是否显示 `active (running)`；
2. 端口是否被 GOST 监听；
3. 云服务商安全组是否已放行端口；
4. VPS 系统防火墙是否已放行端口；
5. 客户端填写的 IP、端口、用户名和密码是否一致。

## 转发与隧道功能概览

菜单 `7` 除了代理功能，还提供：

- TCP + UDP 不加密转发；
- TLS、WS、WSS 加密隧道转发；
- 在落地机解密并转发；
- 多落地简单均衡负载；
- CDN 自选节点转发；
- 自定义 TLS 证书。

中转机和落地机必须选择相互匹配的传输类型，并确保两端相关端口均已放行。涉及 TLS/WSS 时，建议使用有效的自定义证书，不要依赖默认测试证书。

## 独立维护说明

本维护版将与仓库内容相关的地址集中在 `gost.sh` 顶部：

```bash
repo_owner="${GOST_REPO_OWNER:-kjxv}"
repo_name="${GOST_REPO_NAME:-Multi-EasyGost}"
repo_ref="${GOST_REPO_REF:-v2}"
```

脚本安装时使用的 `gost.service`、`config.json`，脚本自更新地址和菜单帮助链接都会基于这些变量生成，不再依赖原作者仓库。以后迁移仓库或测试其他分支时，可以修改默认值，也可以临时覆盖：

```bash
GOST_REPO_OWNER=你的用户名 GOST_REPO_NAME=你的仓库名 GOST_REPO_REF=你的分支 /root/gost.sh
```

GOST 二进制本身仍从 [GOST 官方 Release](https://github.com/ginuerzh/gost/releases) 下载；只有在安装时主动选择大陆镜像，才会改用脚本内标明的第三方二进制镜像。

## 安全与使用注意事项

- 本脚本需要 root 权限，会安装二进制、修改 `/etc/gost`、写入 systemd 服务并重启服务；运行前请先审查脚本。
- 不要使用弱密码，也不要把代理端口无限制暴露给不可信用户。建议结合安全组限制来源 IP。
- 用户名、密码或端口中尽量不要使用会影响 URL/配置解析的特殊字符；密码应足够长且随机。
- 当前底层版本固定为 **GOST v2.11.2**。稳定并不等于始终安全；该版本较旧，请自行关注安全公告、系统日志和异常流量。
- 脚本更新会从本仓库下载并覆盖当前脚本。仓库维护者应先审查、测试，再发布新版本并递增 `shell_version`。
- 搭建代理前请确认符合 VPS 服务商条款和所在地法律法规，不要用于未授权访问、滥发流量或其他违法行为。

## 来源与许可

- GOST 项目：[ginuerzh/gost](https://github.com/ginuerzh/gost)
- 上游脚本：[KANIKIG/Multi-EasyGost](https://github.com/KANIKIG/Multi-EasyGost)
- 原 README 还感谢“风萧萧兮易水寒”的原始脚本以及 STSDUST 的 EasyGost 脚本；本维护版继续保留这份来源说明。
- 本仓库继续采用 [GNU General Public License v3.0](LICENSE)。分发修改版本时必须遵守该许可证，并保留必要的版权与来源说明。

本仓库是社区维护脚本，不是 GOST 官方项目。
