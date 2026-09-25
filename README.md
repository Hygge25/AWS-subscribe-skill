# AWS Lightsail Clash 订阅 Skill

一个面向 Codex 的个人代理部署与维护 Skill，用于在 AWS Lightsail Debian 实例上创建、迁移和排查 Clash Verge 可用的 VLESS Reality 与 Hysteria2 订阅。

它不仅负责生成节点，还覆盖可信 HTTPS 订阅、IP 证书续期、国内外分流、出口信誉诊断、月度流量标签和真实连接验证。

> 本项目不会保存真实服务器 IP、SSH 私钥、UUID、Reality 私钥、Hysteria2 密码或订阅密钥。部署时产生的凭据应只保存在用户自己的设备和服务器中。

## 适用场景

- 已有一台 AWS Lightsail Debian VPS，希望生成 Clash Verge 订阅。
- 同时部署 VLESS Reality（TCP 443）和 Hysteria2（UDP 443）。
- 修复节点 `timeout`、断流、Windows 无法访问或 UDP 不稳定问题。
- VPS 公网 IP 变化后迁移证书、服务和订阅。
- 排查 Google Scholar `automated queries` 等出口 IP 信誉问题。
- 配置国内流量直连、国外流量代理。
- 在 Clash Verge 中显示每月累计流量的 HY2/UDP 标签节点。

不适用于第三方机场管理、通用 AWS 运维，或未经授权的服务器操作。

## 核心能力

- VLESS + Reality + XTLS Vision，使用 TCP 443。
- Hysteria2 + QUIC，使用 UDP 443，与 Reality 共用端口号但不冲突。
- 生成 Mihomo/Clash Verge YAML 和带高熵路径的私有 HTTPS 订阅。
- 使用可信 TLS 证书，不以 `skip-cert-verify: true` 代替证书配置。
- 为短期 IP 证书配置自动续期、证书复制和服务重载。
- 通过独立 Mihomo 实例强制测试每个节点，不切换用户当前活动配置。
- 使用 `GEOSITE,cn`、`GEOIP,CN` 和私网规则实现国内直连。
- 使用 vnStat 持久统计月度流量，并生成真实可连接的 HY2 流量标签节点。
- 区分代理协议故障、客户端配置问题、ISP 路由问题和 AWS 出口 IP 信誉问题。

## 默认架构

| 组件 | 协议 | 端口 | 用途 |
| --- | --- | --- | --- |
| Xray Reality | TCP | 443 | 兼容性较好的主力或回退节点 |
| Hysteria2 | UDP | 443 | 适合丢包链路的 QUIC 节点 |
| Nginx | TLS/TCP | 80 或自定义端口 | 提供私有 Clash 订阅 |
| SSH | TCP | 22 | 服务器维护，建议限制来源地址 |

TCP 443 和 UDP 443 是不同的传输层监听，可以同时存在。实际速度由本地 ISP、到日本的路由、丢包率和 Lightsail 套餐共同决定，不能保证某一种协议在所有网络中都最快。

## 国内外分流

生成的配置采用 `rule` 模式：

1. 局域网与私有地址直连。
2. 中国域名和中国 IP 直连。
3. 未命中的国际流量进入代理策略组。

DNS 策略与路由保持一致，国内域名优先使用国内解析器。验证时会检查 Mihomo 日志，确认国内请求实际命中 `DIRECT`，国际请求实际命中代理，而不是仅检查网站能否打开。

## 安装

将仓库克隆到 Codex Skills 目录：

```bash
git clone https://github.com/Hygge25/AWS-subscribe-skill.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/aws-lightsail-proxy-subscription"
```

重新启动或刷新 Codex 后，Skill 名称为：

```text
aws-lightsail-proxy-subscription
```

已有 Git 安装可使用以下命令更新：

```bash
git -C "${CODEX_HOME:-$HOME/.codex}/skills/aws-lightsail-proxy-subscription" pull --ff-only
```

## 使用示例

部署新订阅：

```text
使用 aws-lightsail-proxy-subscription，为我的 Lightsail Debian 服务器部署
Reality TCP 443 和 Hysteria2 UDP 443，并生成 Clash Verge HTTPS 订阅。
```

排查连接问题：

```text
使用 aws-lightsail-proxy-subscription，检查 HY2 为什么能显示延迟但不能上网。
不要切换我当前正在使用的 Clash 配置。
```

迁移公网 IP：

```text
Lightsail 的公网 IP 已变化。使用 aws-lightsail-proxy-subscription 更新证书、
服务端配置和订阅，并验证自动续期。
```

添加国内直连和月度流量显示：

```text
为当前订阅增加国内直连、国外代理，并添加每月累计流量的 HY2/UDP 标签节点。
```

向 Codex 提供 SSH 用户名、服务器地址和私钥文件路径即可。不要把私钥正文粘贴进对话、README、Skill 或 Git 仓库。

## 工作原则

- 变更服务器前确认用户授权，并保留 SSH 可用性。
- 修改前创建可恢复备份，验证配置后再重启服务。
- 不为了省事关闭 TLS 验证或公开明文订阅。
- 不在诊断时切换用户正在使用的 Clash Verge 配置。
- 不把第三方代理静默加入分流来掩盖 AWS 出口信誉问题。
- 所有结论尽量通过实际连接、Mihomo 日志和服务状态验证。

## 项目结构

```text
.
├── SKILL.md                    # Skill 入口、适用范围和决策原则
├── agents/
│   └── openai.yaml            # Codex 界面元数据
└── references/
    └── aws-lightsail.md        # 部署、迁移、分流与排障运行手册
```

详细技术流程请阅读 [AWS Lightsail 运行手册](references/aws-lightsail.md)。

## 已知限制

- 普通 Lightsail 公网 IP 可能在停止/启动后变化；稳定使用建议评估并绑定静态 IP。
- AWS 数据中心 IP 可能被部分网站限流或识别为自动请求，协议调优无法改变 IP 信誉。
- vnStat 标签统计的是 VPS 公网接口总流量，包含代理流量以及少量 SSH、证书续期和订阅下载流量。
- Hysteria2 并不必然比 Reality 快，应从实际客户端网络进行多轮对比。

## 安全与合规

本项目用于管理用户本人拥有或获授权管理的服务器。请遵守所在地法律、AWS 服务条款和目标网站的使用政策。订阅链接本身包含访问凭据，应像密码一样保存，不要发布到公开仓库、截图或日志中。
