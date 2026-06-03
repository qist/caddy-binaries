# Caddy Binaries with L4 Plugin

[![Build Status](https://github.com/qist/caddy-binaries/actions/workflows/check-and-build.yml/badge.svg)](https://github.com/qist/caddy-binaries/actions/workflows/check-and-build.yml)

预编译的 Caddy 二进制文件，包含 [caddy-l4](https://github.com/mholt/caddy-l4) 插件支持。

## 功能特点

- ✅ 自动跟踪官方 Caddy 最新版本
- ✅ 集成 caddy-l4 插件（Layer 4 TCP/UDP 代理）
- ✅ 多平台支持
- ✅ 每日自动检查更新
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

## 使用方法

### Linux/macOS

```bash
# 解压（直接得到二进制文件）
tar -xzf caddy_v2.9.0_linux_amd64.tar.gz

# 运行
chmod +x caddy
./caddy version
./caddy run --config Caddyfile
```

### Windows

```powershell
# 解压（直接得到二进制文件）
Expand-Archive caddy_v2.9.0_windows_amd64.zip -DestinationPath .

# 运行
.\caddy version
.\caddy run --config Caddyfile
```

## caddy-l4 插件

caddy-l4（Project Conncept）是一个 Layer 4 TCP/UDP 应用插件，允许 Caddy 代理原始 TCP/UDP 连接，扩展了传统的 HTTP 代理功能。

### 配置示例

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

更多配置示例请参考 [caddy-l4 官方文档](https://github.com/mholt/caddy-l4)。

### SNI 分流配置示例

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

## 自动构建

本仓库使用 GitHub Actions 自动构建：

1. **每日检查**：每天 UTC 00:00 自动检查官方 Caddy 最新版本
2. **版本对比**：对比已构建的版本，避免重复构建
3. **自动发布**：检测到新版本时自动编译并发布到 Releases

## 构建流程

```
┌─────────────────────────────────────────────────────────┐
│  check-and-build.yml (每日定时检查)                      │
├─────────────────────────────────────────────────────────┤
│  1. 获取官方 Caddy 最新 tag                              │
│  2. 对比本地 built_tag.txt                              │
│  3. 如果版本不同，触发 build.yml                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  build.yml (编译工作流)                                  │
├─────────────────────────────────────────────────────────┤
│  1. 安装 Go 环境                                        │
│  2. 使用 xcaddy 编译 (包含 caddy-l4)                    │
│  3. 生成多平台二进制文件                                 │
│  4. 创建压缩包                                          │
│  5. 发布到 GitHub Releases                              │
│  6. 更新 built_tag.txt                                  │
└─────────────────────────────────────────────────────────┘
```

## 许可证

Caddy 采用 Apache 2.0 许可证，详见 [LICENSE](https://github.com/caddyserver/caddy/blob/master/LICENSE)。

## 相关链接

- [Caddy 官方网站](https://caddyserver.com/)
- [Caddy GitHub](https://github.com/caddyserver/caddy)
- [caddy-l4 插件](https://github.com/mholt/caddy-l4)