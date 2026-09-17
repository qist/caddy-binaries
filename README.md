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
- ✅ WAF 全规则开启仅 ~1.7% 性能开销

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
    ├── useragent.rule     # 攻击扫描器 / 渗透工具 UA 黑名单
    ├── header.rule        # 请求头黑名单（绕过类头/SSRF 元数据头/Log4Shell）
    ├── referer.rule       # 恶意 Referer 黑名单
    ├── whiteip.rule       # IP 白名单
    ├── whiteua.rule       # User-Agent 白名单
    ├── whiteurl.rule      # URL 白名单
    ├── blackip.rule       # IP 黑名单
    ├── cdnip.rule         # CDN/可信代理 IP 列表（控制 XFF 信任，支持 CIDR）
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
- **13 项检测链**：白名单 IP/URL/UA、黑名单 IP、CC 攻击防护、URL 路径/参数检测（含 256+ 参数截断兜底）、User-Agent/请求头/Cookie/Referer 检测、POST body 检测（含大 body 超限拦截 + 实体拆分兜底）、文件上传扩展名检测
- **IPv4/IPv6 双栈**：IP 黑白名单同时支持 IPv4 和 IPv6，支持 CIDR 表示法（`192.168.1.0/24`、`2001:db8::/32`）、glob 通配符（`192.168.*.*`、`2001:db8::*`）和精确匹配
- **高性能**：正则预编译（含 `(?i)` 大小写不敏感版本）+ POST body 关键词自动提取预过滤 + 64 分片 CC 存储 + Config 预合并缓存，WAF 全规则开启仅 ~1.7% 性能开销
- **热加载**：规则和配置文件修改后 2 秒内自动生效，无需重启 Caddy
- **域名级配置**：支持全局配置 + 按域名覆盖（精确匹配 + 通配符）+ 域名级独立规则目录 + 域名级扫描阈值覆盖
- **路径级配置**：支持基于 Caddy 原生 `path` matcher 的 WAF 开关，可对特定 URL 路径关闭 WAF
- **Body 扫描控制**：`post_body_scan_limit` 超限直接拦截防部分扫描误放行；`multipart_streaming_check` 开关控制 multipart body 是否走流式扫描；`upload_filename_scan_limit` 控制文件名扫描范围
- **bodyless 方法控制**：`bodyless` 配置项控制 GET/HEAD/OPTIONS 是否跳过 body 检测，支持域名级覆盖（如对特定域名强制全方法扫描）
- **同步日志**：与 Lua 版一致，攻击日志同步落盘，不丢失。`sync.Mutex` 保护并发写入。日志字段自动截断到 4096 字节，防止单条日志过大
- **cc_rate 配置校验**：无效的 `cc_rate` 配置自动记录错误日志，避免 CC 防护静默失效
- **CDN 代理 IP 信任**：`cdnip.rule` 控制 XFF 信任范围，只有来自可信 CDN/代理 IP 的请求才信任 X-Forwarded-For，防直连伪造
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
├── url.rule             # URL 路径黑名单
├── args.rule            # URL 参数黑名单（SQL注入/XSS/SSTI/RCE等）
├── post.rule            # POST body 黑名单
├── cookie.rule          # Cookie 黑名单
├── useragent.rule       # 攻击扫描器 / 渗透工具 UA 黑名单
├── header.rule          # 请求头黑名单（绕过类头/SSRF 元数据头/Log4Shell）
├── referer.rule         # 恶意 Referer 黑名单（支付接口保护）
├── whiteip.rule         # IP 白名单
├── whiteua.rule         # User-Agent 白名单（搜索引擎蜘蛛）
├── whiteurl.rule        # URL 白名单
├── blackip.rule         # IP 黑名单
├── cdnip.rule           # CDN/可信代理 IP 列表（控制 XFF 信任，支持 CIDR）
├── fileext.rule         # 文件上传扩展名黑名单
└── domains/             # 域名级独立规则目录
    └── www.example.com/ # 该域名专用规则（12 个 .rule 文件，未提供的文件回退全局）
        ├── url.rule
        ├── args.rule
        ├── post.rule
        ├── cookie.rule
        ├── useragent.rule
        ├── header.rule
        ├── whiteua.rule
        ├── referer.rule
        ├── fileext.rule
        ├── whiteip.rule
        ├── whiteurl.rule
        └── blackip.rule
```

#### config.json 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `waf_enable` | `"on"` | WAF 总开关，`"off"` 完全关闭 |
| `trust_proxy_headers` | `"on"` | 是否信任代理转发的 IP 头（X-Forwarded-For 等）。`"on"`=根据 `cdnip.rule` 判断是否信任转发头；`"off"`=只用 `remote_addr` 防伪造 |
| `log_dir` | `/var/log/caddyguard` | WAF 日志目录 |
| `url_check` | `"on"` | URL 路径检测开关 |
| `url_args_check` | `"on"` | URL 参数检测开关 |
| `post_check` | `"on"` | POST body 检测开关 |
| `user_agent_check` | `"on"` | User-Agent 检测开关 |
| `header_check` | `"on"` | 请求头检测开关（`header.rule`：绕过类头 / SSRF 元数据头 / Log4Shell） |
| `cookie_check` | `"on"` | Cookie 检测开关 |
| `cc_check` | `"on"` | CC 攻击防护开关 |
| `cc_rate` | `"60/60"` | CC 速率限制，格式 `请求数/时间窗口秒`。无效配置自动记录错误日志并禁用 CC 检测 |
| `cc_block_ttl` | `600` | CC 触发后封禁时长（秒） |
| `white_ip_check` | `"on"` | IP 白名单检测开关 |
| `white_ua_check` | `"on"` | UA 白名单检测开关 |
| `white_url_check` | `"on"` | URL 白名单检测开关 |
| `black_ip_check` | `"on"` | IP 黑名单检测开关 |
| `referer_check` | `"off"` | Referer 检测开关 |
| `file_upload_check` | `"on"` | 文件上传扩展名检测开关 |
| `bodyless` | `"on"` | bodyless 方法跳过开关。`"on"`=GET/HEAD/OPTIONS 跳过 body/post/file_upload 检测；`"off"`=所有方法都扫描 body |
| `multipart_streaming_check` | `"off"` | multipart body 流式内容扫描开关。默认 `off`，关闭时仍保留文件名扩展名检查 |
| `upload_filename_scan_limit` | `0` | multipart 文件名扫描上限（字节）。`0`=扫描整个文件；正整数=只扫描前 N 字节 |
| `post_body_scan_limit` | `2097152` | 非 multipart body 扫描上限（字节）。超过此值直接拦截 |
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

# useragent.rule — 攻击扫描器 / 渗透工具 UA（命中 403）
# 短词必须加边界，避免误杀正常业务 UA
(HTTrack|harvest|pangolin|nmap|sqlmap|w3af|fimap|havij|PycURL|netsparker|httperf|ApacheBench|wrk/|hey/|k6/)
(Acunetix|WebVulnScan|Paros|WebInspect|\bBurp\b|BurpSuite|AppScan|Arachni|Skipfish|Wapiti|WhatWeb|Wfuzz|DirBuster|GoBuster|ffuf|dirmap|feroxbuster|Nuclei|subfinder|masscan|ZGrab|Shodan|Censys|wpscan|nikto|dirb|pwntools)
((?:^|[^-\w])httpx/|httpx-cli\b)         # 不匹配 python-httpx/、httpx-client
(?:^|[^-\w])amass(?:$|[^-\w])            # 不匹配 amass-client
(OWASP ZAP|ZAP/)                         # 不匹配 OWASP-Dependency-Check
(xray/|afrog|fscan|TscanPlus|Yakit|W13Scan|vulmap|PocSuite|BBOT|katana/|\bGoby\b)
# 正常爬虫（Amazonbot/Applebot-Extended/ia_archiver）与调试工具（Postman/Charles/Fiddler）不拦截

# header.rule — 请求头黑名单（多行匹配，^ 锚定头名）
(?i:^x-middleware-subrequest:)           # Next.js CVE-2025-29927 中间件绕过
(?i:^x-original-url:)
(?i:^x-rewrite-url:)
(?i:\$\{jndi:)                           # 请求头中的 Log4Shell / JNDI 注入

# fileext.rule — 文件上传扩展名黑名单
\.php\..*\.(htaccess|bash_history)
\.(htaccess|bash_history|htpasswd|gitignore|gitattributes|env|config|sql|bak|backup|old|tmp|log|swp|sql\.gz)

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

#### IP 规则文件格式

`whiteip.rule`、`blackip.rule` 和 `cdnip.rule` 每行一条 IP 规则，支持三种格式：

```
# 1. CIDR 表示法（推荐，IPv4/IPv6 均支持）
192.168.1.0/24          # IPv4 CIDR
2001:db8::/32           # IPv6 CIDR
10.0.0.0/8              # IPv4 大范围
::1/128                 # IPv6 loopback

# 2. glob 通配符
192.168.1.*             # IPv4 通配符
2001:db8::*             # IPv6 通配符
192.168.*.*             # 多段通配符

# 3. 精确匹配
8.8.8.8                 # IPv4 精确
2001:db8::5             # IPv6 精确
::1                     # IPv6 loopback
```

#### CDN 代理 IP 信任（cdnip.rule）

当 CaddyGuard 部署在 CDN/反向代理后面时，需要从 `X-Forwarded-For` 获取真实客户端 IP。但直接信任 XFF 会让攻击者伪造该头绕过 IP 黑白名单。

解决方案：在 `config.json` 中设置 `trust_proxy_headers = "on"`，并在 `cdnip.rule` 中配置你实际使用的 CDN/代理 IP 段：

```json
// config.json
{
    "trust_proxy_headers": "on"
}
```

```bash
# cdnip.rule
# 填入你实际使用的 CDN/代理 IP 段，以下为示例
# Cloudflare IPv4
173.245.48.0/20
104.16.0.0/13
# Cloudflare IPv6
2400:cb00::/32
2606:4700::/32
# 内部代理/负载均衡器
10.0.0.0/8
192.168.0.0/16
```

此时 CaddyGuard 的行为：

| 条件 | XFF 处理 | 说明 |
|------|---------|------|
| `remote_addr` 在 cdnip.rule 中 | 信任 XFF | 提取真实客户端 IP |
| `remote_addr` 不在 cdnip.rule 中 | 不信任 XFF | 使用 `remote_addr`（防直连伪造） |
| `cdnip.rule` 文件不存在 | 信任所有 XFF | 原始行为，向后兼容 |
| `cdnip.rule` 文件为空 | 信任所有 XFF | 等同于文件不存在 |

#### 三种白名单的区别

| 白名单 | 文件 | 行为 | 说明 |
|--------|------|------|------|
| **白名单 IP** | `whiteip.rule` | **全局放行**，跳过全部 13 项检测 | 信任 IP，完全不做任何安全检测 |
| **白名单 URL** | `whiteurl.rule` | **仅跳过指定检测项**（默认只跳过 URL 路径检测），其他检测照常 | 可配置跳过哪些检测项（`user_agent`/`header`/`referer`/`url_attack`/`url_args`/`cookie`/`post`/`file_upload`/`cc`） |
| **白名单 UA** | `whiteua.rule` | **仅跳过 UA 黑名单检测**，其他检测照常 | 搜索引擎蜘蛛免被 UA 黑名单误杀，但仍受 URL/参数/请求头/POST 等检测约束 |

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
    "limit.example.com": {
        "_comment": "示例：域名级 body/file 扫描阈值覆盖",
        "multipart_streaming_check": "on",
        "post_body_scan_limit": 1048576,
        "upload_filename_scan_limit": 1024
    },
    "strict.example.com": {
        "_comment": "示例：对该域名强制扫描所有方法的 body（包括 GET/HEAD/OPTIONS）",
        "bodyless": "off"
    },
    "*.test.com": {
        "post_check": "off",
        "cookie_check": "off"
    }
}
```

- **精确域名**：`www.example.com` → O(1) map 查找
- **通配符域名**：`*.example.com` → 加载时预解析为列表，按后缀匹配
- **域名级规则目录**：`rule_dir` 指定域名专用规则目录，该域名请求使用独立规则文件覆盖全局规则

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

### 插件版本固定（避免打到旧代码）

插件不再使用不带版本的 `--with`（那会解析 `@latest`，命中 module proxy / runner 缓存时会拿到旧 commit），而是：

1. `check-version` 阶段用 GitHub API 解析 **caddyguard / caddy-l4 的最新 commit SHA**，所有矩阵任务共用同一个 SHA
2. 构建时固定到该 commit：`--with github.com/qist/caddyguard@<sha>`（pseudo-version 不可变，Go 模块缓存不会导致旧代码）
3. 插件模块走 git 直连（`GONOPROXY`）绕过 proxy 缓存
4. 构建后用 `go version -m` 校验产物里嵌入的 commit 与预期一致，不一致直接失败，不发布
5. 发布说明中记录两个插件的 commit，可用 `go version -m caddy | grep caddyguard` 自行核对

### 插件更新后重新打包

`git tag` 已存在时默认跳过构建；插件（caddyguard / nginxguard 规则）有更新需要重新出包时，
手动触发 workflow 并勾选 `force_build`（或指定新的 `caddy_tag`）：

```
Actions → Auto Build Caddy with L4 Plugin → Run workflow
  caddy_tag:  latest（或指定版本）
  force_build: true
```

> 规则文件（`rule-config/`）在构建时从 [nginxguard](https://github.com/qist/nginxguard) 仓库实时克隆，
> 所以**规则改动只要推送到 nginxguard 就会进下次打包**；`config.json` 由 CI 生成，新增配置项需同步修改 `build.yml`。

## 构建流程

```
┌─────────────────────────────────────────────────────────┐
│  build.yml (单一工作流)                                  │
├─────────────────────────────────────────────────────────┤
│  1. check-version: 获取官方 Caddy 最新 tag              │
│  2. check-version: 解析 caddyguard / caddy-l4 commit    │
│  3. check-version: 检查 git tag 是否已存在（支持 force） │
│  4. 如果 tag 不存在（或 force_build），触发多平台并行编译 │
│  5. 使用 xcaddy 编译 (固定插件 commit) + 校验产物 commit │
│  6. 生成多平台二进制文件并创建压缩包                     │
│  7. release: 发布到 GitHub Releases 并创建 git tag       │
└─────────────────────────────────────────────────────────┘
```

## 许可证

Caddy 采用 Apache 2.0 许可证，详见 [LICENSE](https://github.com/caddyserver/caddy/blob/master/LICENSE)。

## 相关链接

- [Caddy 官方网站](https://caddyserver.com/)
- [Caddy GitHub](https://github.com/caddyserver/caddy)
- [caddy-l4 插件](https://github.com/mholt/caddy-l4)
- [caddyguard 插件](https://github.com/qist/caddyguard)