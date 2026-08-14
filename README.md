# Caddy Binaries with L4 + WAF Plugins

[![Build Status](https://github.com/qist/caddy-binaries/actions/workflows/build.yml/badge.svg)](https://github.com/qist/caddy-binaries/actions/workflows/build.yml)

预编译的 Caddy 二进制文件，包含 [caddy-l4](https://github.com/mholt/caddy-l4) 和 [caddyguard](https://github.com/qist/caddyguard) 插件支持。

## 功能特点

- ✅ 自动跟踪官方 Caddy 最新版本
- ✅ 集成 caddy-l4 插件（Layer 4 TCP/UDP 代理）
- ✅ 集成 caddyguard 插件（安全防护/WAF）
- ✅ 多平台支持
- ✅ 每日自动检查更新
- ✅ 基于 git tag 判断版本，避免重复构建
- ✅ 官方格式的压缩包

## 支持的平台

| 操作系统 | 架构 | 文件格式 |
|----------|------|----------|
| Linux | amd64 | `.tar.gz` |
| Linux | arm64 | `.tar.gz` |
| Linux | armv7 | `.tar.gz` |
| Windows | amd64 | `.zip` |
| Windows | arm64 | `.zip` |
| macOS | amd64 | `.tar.gz` |
| macOS | arm64 | `.tar.gz` |
| FreeBSD | amd64 | `.tar.gz` |
| FreeBSD | arm64 | `.tar.gz` |

## 下载

前往 [Releases](https://github.com/qist/caddy-binaries/releases) 页面下载最新版本。

### 文件命名格式

```
caddy_<version>_<os>_<arch>.tar.gz  # Linux/macOS/FreeBSD
caddy_<version>_<os>_<arch>.zip      # Windows
```

示例：
- `caddy_v2.9.0_linux_amd64.tar.gz`
- `caddy_v2.9.0_windows_amd64.zip`
- `caddy_v2.9.0_darwin_arm64.tar.gz`

## 压缩包内容

每个平台的压缩包中包含以下内容：

```
caddy_v2.9.0_linux_amd64.tar.gz
├── caddy                  # 二进制文件
└── caddyguard/            # WAF 规则配置文件
    ├── config.json        # 全局 WAF 配置
    ├── domain.json        # 域名级配置覆盖
    ├── url.rule           # URL 路径黑名单
    ├── args.rule          # URL 参数黑名单
    ├── post.rule          # POST body 黑名单
    ├── cookie.rule        # Cookie 黑名单
    ├── useragent.rule     # 恶意 User-Agent 黑名单
    ├── referer.rule       # 恶意 Referer 黑名单
    ├── whiteip.rule       # IP 白名单
    ├── whiteua.rule       # User-Agent 白名单
    ├── whiteurl.rule      # URL 白名单
    ├── blackip.rule       # IP 黑名单
    ├── fileext.rule       # 文件上传扩展名黑名单
    └── domains/           # 域名级独立规则目录
```

## 使用方法

### Linux/macOS

```bash
# 解压（得到二进制文件 + 规则配置目录）
tar -xzf caddy_v2.9.0_linux_amd64.tar.gz

# 运行（使用 caddyguardfile 适配器，WAF 全局自动生效）
chmod +x caddy
./caddy version
./caddy run --config Caddyfile --adapter caddyguardfile
```

### Windows

```powershell
# 解压（得到二进制文件 + 规则配置目录）
Expand-Archive caddy_v2.9.0_windows_amd64.zip -DestinationPath .

# 运行（使用 caddyguardfile 适配器，WAF 全局自动生效）
.\caddy version
.\caddy run --config Caddyfile --adapter caddyguardfile
```

> **注意**：如果使用 caddyguard WAF 功能，请将 `caddyguard/` 目录复制到 `/etc/caddyguard/rule-config`，或在 Caddyfile 中指定实际路径。使用 `--adapter caddyguardfile` 启动可全局自动生效，无需在每个站点单独配置。

## 包含的插件

### caddy-l4 插件

caddy-l4（Project Conncept）是一个 Layer 4 TCP/UDP 应用插件，允许 Caddy 代理原始 TCP/UDP 连接，扩展了传统的 HTTP 代理功能。

#### 配置示例

```caddyfile
{
    layer4 {
        # TCP 端口转发：将 2222 端口的流量转发到 localhost:22
        :2222 {
            route {
                proxy localhost:22
            }
        }
        
        # TLS 终止后转发
        :8443 {
            route {
                tls
                proxy localhost:8080
            }
        }
    }
}

# 同时可以配置 HTTP 服务
localhost:80 {
    respond "Hello, World!"
}
```

#### SNI 分流配置示例

基于 TLS SNI (Server Name Indication) 进行 Layer 4 分流：

```caddyfile
{
    layer4 {
        # 监听 443 端口，根据 SNI 进行分流
        :443 {
            route {
                # 如果 SNI 匹配 ssh.example.com，转发到 SSH 服务
                @ssh sni ssh.example.com
                handle @ssh {
                    proxy localhost:22
                }
                
                # 如果 SNI 匹配 git.example.com，转发到 Git 服务
                @git sni git.example.com
                handle @git {
                    proxy localhost:2222
                }
                
                # 默认：转发到 HTTPS 服务
                proxy localhost:443
            }
        }
    }
}
```

**说明**：
- 使用 `sni` 匹配器根据 TLS 握手时的 Server Name 进行路由
- `handle` 块按照顺序匹配，第一个匹配的规则生效
- 支持同时代理多种服务（SSH、Git、HTTPS 等）到同一端口

更多配置示例请参考 [caddy-l4 官方文档](https://github.com/mholt/caddy-l4)。

### caddyguard 插件

caddyguard 是一个 Caddy 安全防护插件，提供 WAF（Web 应用防火墙）功能，包括：

- **全局自动生效**：全局配置一次 `rule_dir`，所有站点自动启用 WAF，无需每个站点单独写 `caddyguard` 指令（通过 `caddyguardfile` 适配器实现）
- **12 项检测链**：白名单 IP/URL/UA、黑名单 IP、CC 攻击防护、URL 路径/参数检测、User-Agent/Cookie/Referer 检测、POST body 检测、文件上传扩展名检测
- **IPv4/IPv6 双栈**：IP 黑白名单同时支持 IPv4 和 IPv6，支持 CIDR、glob 通配符和精确匹配
- **高性能**：正则预编译（含 `(?i)` 大小写不敏感版本）+ POST body 关键词自动提取预过滤 + 64 分片 CC 存储 + Config 预合并缓存，WAF 全规则开启仅 ~11% 性能开销
- **热加载**：规则和配置文件修改后 2 秒内自动生效，无需重启 Caddy
- **域名级配置**：支持全局配置 + 按域名覆盖（精确匹配 + 通配符）+ 域名级独立规则目录
- **路径级配置**：支持基于 Caddy 原生 `path` matcher 的 WAF 开关，可对特定 URL 路径关闭 WAF
- **零 reflect/unsafe**：使用 Caddy 标准中间件链，不依赖私有字段反射
- **ReDoS 安全**：基于 Go RE2 正则引擎，无回溯爆炸风险

#### 配置方式

CaddyGuard 提供三种配置方式：

##### 方式 1：全局配置 + `caddyguardfile` 适配器（推荐）

全局配置一次 `rule_dir`，所有站点自动启用 WAF，**站点块不需要写 `caddyguard`**。
`caddyguardfile` 适配器在解析 Caddyfile 后自动向每个 HTTP server 注入 Guard handler。

```caddyfile
{
    auto_https off

    # 全局 WAF 配置 — 只写一次
    caddyguard {
        rule_dir /etc/caddyguard/rule-config
    }
}

# 站点不需要写 caddyguard，自动生效
example.com {
    reverse_proxy 127.0.0.1:8080
}

another.com {
    reverse_proxy 127.0.0.1:9090
}
```

启动时使用 `--adapter caddyguardfile`：

```bash
caddy run --config /etc/caddy/Caddyfile --adapter caddyguardfile
```

##### 方式 2：站点级配置 + 标准 `caddyfile` 适配器

每个站点单独写 `caddyguard` 指令，适合需要精细控制的场景。

```caddyfile
{
    auto_https off
}

example.com {
    caddyguard {
        rule_dir /etc/caddyguard/rule-config
    }
    reverse_proxy 127.0.0.1:8080
}
```

启动时使用标准 `caddyfile` 适配器（默认）：

```bash
caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
```

##### 方式 3：JSON 配置（适合自动化部署 / Docker / K8s）

直接使用 Caddy 原生 JSON 配置，无需 adapter。WAF handler 需手动写在每个 route 的 `handle` 列表最前面。

```json
{
    "apps": {
        "caddyguard": {
            "rule_dir": "/etc/caddyguard/rule-config"
        },
        "http": {
            "servers": {
                "srv0": {
                    "automatic_https": { "disable": true },
                    "listen": [":80"],
                    "routes": [
                        {
                            "handle": [
                                { "handler": "caddyguard" },
                                { "handler": "reverse_proxy", "upstreams": [{ "dial": "127.0.0.1:8080" }] }
                            ]
                        }
                    ]
                }
            }
        }
    }
}
```

启动时不需要 `--adapter` 参数（默认即为 JSON）：

```bash
caddy run --config /etc/caddy/caddy.json
```

**关键点**：
- `apps.caddyguard.rule_dir` 指定规则目录，与 Caddyfile 方式等效
- 每个 route 的 `handle` 列表中，`{"handler": "caddyguard"}` 必须写在其他 handler 前面
- 路径级 WAF 开关：`{"handler": "caddyguard", "waf_enable": "off"}`
- WAF 检测开关仍由 `rule_dir/config.json` 控制

##### 三种方式对比

| | 方式 1：caddyguardfile | 方式 2：caddyfile | 方式 3：JSON |
|---|---|---|---|
| 全局配置 | ✅ 一次配置，所有站点自动生效 | ❌ 每个站点需单独写 | ❌ 每个 route 需手动写 |
| 站点块 | 只写业务指令 | 需写 `caddyguard` 指令 | 需写 `caddyguard` handler |
| 启动参数 | `--adapter caddyguardfile` | `--adapter caddyfile`（默认） | 不需要 adapter |
| 站点级覆盖 | 支持（站点写 `caddyguard { rule_dir ... }`） | 支持 | 支持（route 级 `waf_enable`） |
| 路径级开关 | ✅ Caddy 原生 path matcher | ✅ Caddy 原生 path matcher | ✅ route 级 `waf_enable: off` |
| 自动注入 | ✅ adapter 自动注入 handler | ❌ 需手动写 `caddyguard` 指令 | ❌ 需手动写 handler |
| 推荐场景 | 多站点统一 WAF | 单站点或精细控制 | 自动化部署 / Docker / K8s |

#### 规则目录结构

```
/etc/caddyguard/rule-config/
├── config.json          # 全局 WAF 配置
├── domain.json          # 域名级配置覆盖
├── url.rule             # URL 路径黑名单（92 条规则）
├── args.rule            # URL 参数黑名单（95 条规则，SQL注入/XSS/SSTI/RCE等）
├── post.rule            # POST body 黑名单（96 条规则）
├── cookie.rule          # Cookie 黑名单（96 条规则）
├── useragent.rule       # 恶意 User-Agent 黑名单（5 条规则，扫描器/爬虫）
├── referer.rule         # 恶意 Referer 黑名单（8 条规则，支付接口保护）
├── whiteip.rule         # IP 白名单
├── whiteua.rule         # User-Agent 白名单（44 条，搜索引擎蜘蛛）
├── whiteurl.rule        # URL 白名单
├── blackip.rule         # IP 黑名单
├── fileext.rule         # 文件上传扩展名黑名单
└── domains/             # 域名级独立规则目录
    └── www.example.com/ # 该域名专用规则（8 个 .rule 文件）
        ├── url.rule
        ├── args.rule
        ├── post.rule
        ├── cookie.rule
        ├── useragent.rule
        ├── whiteip.rule
        ├── whiteurl.rule
        └── blackip.rule
```

#### config.json 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `waf_enable` | `"on"` | WAF 总开关，`"off"` 完全关闭 |
| `trust_proxy_headers` | `"on"` | 信任 X-Forwarded-For 头获取客户端 IP |
| `log_dir` | `/var/log/caddyguard` | WAF 日志目录 |
| `url_check` | `"on"` | URL 路径检测开关 |
| `url_args_check` | `"on"` | URL 参数检测开关 |
| `post_check` | `"on"` | POST body 检测开关 |
| `user_agent_check` | `"on"` | User-Agent 检测开关 |
| `cookie_check` | `"on"` | Cookie 检测开关 |
| `cc_check` | `"on"` | CC 攻击防护开关 |
| `cc_rate` | `"60/60"` | CC 速率限制，格式 `请求数/时间窗口秒` |
| `cc_block_ttl` | `600` | CC 触发后封禁时长（秒） |
| `white_ip_check` | `"on"` | IP 白名单检测开关 |
| `white_ua_check` | `"on"` | UA 白名单检测开关 |
| `white_url_check` | `"on"` | URL 白名单检测开关 |
| `black_ip_check` | `"on"` | IP 黑名单检测开关 |
| `referer_check` | `"off"` | Referer 检测开关 |
| `file_upload_check` | `"on"` | 文件上传扩展名检测开关 |
| `waf_output` | `"html"` | 拦截响应模式：`"html"` 返回拦截页面，`"redirect"` 302 跳转 |
| `waf_redirect_url` | - | `waf_output` 为 redirect 时的跳转 URL |

#### 规则文件格式

所有 `.rule` 文件每行一条正则表达式（Go RE2 语法），`#` 开头为注释，空行忽略：

```
# url.rule — URL 路径黑名单
\/wp-login\.php
\/phpinfo\.php
\/\.env
\/\.git\/
(phpmyadmin|jmx-console|admin-console)

# args.rule — URL 参数黑名单
select.+(from|limit)
(?:(union(.*?)select))
sleep\((\s*)(\d*)(\s*)\)
\<(iframe|script|body|img|layer|div|meta|style|base|object|input)

# useragent.rule — 恶意 UA
(HTTrack|harvest|audit|dirbuster|pangolin|nmap|sqlmap|w3af|owasp|Nikto)
(Acunetix|WebVulnScan|Paros|WebInspect|Burp|BurpSuite|WebScarab|Nuclei|httpx)
(Python-urllib|Python-requests|Go-http-client|scrapy|bot|crawl|spider|fetcher)

# fileext.rule — 文件上传扩展名黑名单
\.php
.*\.(htaccess|bash_history|htpasswd|gitignore|gitattributes|env|config|sql|bak|backup|old|tmp|log|swp|sql\.gz)

# referer.rule — 恶意 Referer 黑名单（支付接口保护）
\.pay\.
\.alipay\.
\.tenpay\.
\.paypal\.
\.stripe\.

# whiteua.rule — User-Agent 白名单（搜索引擎蜘蛛，仅跳过 UA 黑名单检测）
Googlebot
Baiduspider
bingbot
360Spider
YandexBot
```

#### 三种白名单的区别

| 白名单 | 文件 | 行为 | 说明 |
|--------|------|------|------|
| **白名单 IP** | `whiteip.rule` | **全局放行**，跳过全部 12 项检测 | 信任 IP，完全不做任何安全检测 |
| **白名单 URL** | `whiteurl.rule` | **全局放行**，跳过全部 12 项检测 | 信任 URL 路径，完全不做任何安全检测 |
| **白名单 UA** | `whiteua.rule` | **仅跳过 UA 黑名单检测**，其他检测照常 | 搜索引擎蜘蛛免被 UA 黑名单误杀，但仍受 URL/参数/POST 等检测约束 |

#### domain.json 域名级覆盖

```json
{
    "www.example.com": {
        "url_check": "off",
        "cc_rate": "100/60",
        "rule_dir": "domains/www.example.com"
    },
    "api.example.com": {
        "waf_enable": "off"
    },
    "*.test.com": {
        "post_check": "off",
        "cookie_check": "off"
    }
}
```

- 精确域名：`www.example.com` → O(1) map 查找
- 通配符域名：`*.example.com` → 加载时预解析为列表，按后缀匹配
- 域名级规则目录：`rule_dir` 指定域名专用规则目录，该域名请求使用独立规则文件覆盖全局规则

更多配置请参考 [caddyguard 官方文档](https://github.com/qist/caddyguard)。

### caddy-l4 + caddyguard 组合配置

两个插件可以在同一个 Caddyfile 中同时使用，全局块中分别配置即可：

```caddyfile
{
    # Layer 4 TCP/UDP 代理
    layer4 {
        :2222 {
            route {
                proxy localhost:22
            }
        }
    }

    # 全局 WAF 配置
    caddyguard {
        rule_dir /etc/caddyguard/rule-config
    }
}

example.com {
    reverse_proxy 127.0.0.1:8080
}
```

> 使用 `--adapter caddyguardfile` 启动时，WAF 会自动对所有 HTTP 站点生效，同时 layer4 配置正常工作。

## 自动构建

本仓库使用 GitHub Actions 自动构建：

1. **每日检查**：每天 UTC 00:00 自动检查官方 Caddy 最新版本
2. **版本对比**：通过 git tag 判断是否已构建过该版本，避免重复构建
3. **自动发布**：检测到新版本时自动编译并发布到 Releases

## 构建流程

```
┌─────────────────────────────────────────────────────────┐
│  build.yml (单一工作流)                                  │
├─────────────────────────────────────────────────────────┤
│  1. check-version: 获取官方 Caddy 最新 tag              │
│  2. check-version: 检查 git tag 是否已存在               │
│  3. 如果 tag 不存在，触发多平台并行编译                  │
│  4. 使用 xcaddy 编译 (包含 caddy-l4 + caddyguard)       │
│  5. 生成多平台二进制文件并创建压缩包                     │
│  6. release: 发布到 GitHub Releases 并创建 git tag       │
└─────────────────────────────────────────────────────────┘
```

## 许可证

Caddy 采用 Apache 2.0 许可证，详见 [LICENSE](https://github.com/caddyserver/caddy/blob/master/LICENSE)。

## 相关链接

- [Caddy 官方网站](https://caddyserver.com/)
- [Caddy GitHub](https://github.com/caddyserver/caddy)
- [caddy-l4 插件](https://github.com/mholt/caddy-l4)
- [caddyguard 插件](https://github.com/qist/caddyguard)