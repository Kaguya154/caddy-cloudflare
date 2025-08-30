# 🚀 Custom Caddy Build · 内置 Cloudflare DNS 插件

[![构建状态](https://github.com/qichiyuhub/caddy-cloudflare/actions/workflows/build-caddy.yml/badge.svg)](https://github.com/qichiyuhub/caddy-cloudflare/actions)
[![最新版本](https://img.shields.io/github/v/release/qichiyuhub/caddy-cloudflare)](https://github.com/qichiyuhub/caddy-cloudflare/releases/latest)

这是一个自动化构建项目，旨在提供集成了官方 **Cloudflare DNS 插件** 的 **Caddy** 二进制文件。您无需自行搭建 Go 编译环境，即可从 [Releases](https://github.com/qichiyuhub/caddy-cloudflare/releases/latest) 页面直接下载用于生产环境、支持 DNS-01 质询的 Caddy。

---

### 解决了什么问题？

官方发布的 Caddy 不包含任何 DNS 插件。若要使用 DNS 提供商（如 Cloudflare）来自动申请泛域名或内网站点的 HTTPS 证书，用户必须自行下载 Caddy 源码和插件源码，并使用 `xcaddy` 进行编译。这个过程不仅繁琐，而且每次 Caddy 更新都需要重复操作。

**本仓库通过 GitHub Actions 将这一过程完全自动化，为您提供一个开箱即用的解决方案。**

### ✨ 核心特性

- **Cloudflare DNS 支持**：内置官方 Cloudflare DNS 插件，轻松实现基于 DNS-01 质询的泛域名及内网域名证书自动签发。
- **跨平台架构**：为 `linux/amd64` 和 `linux/arm64` 提供原生二进制文件，无缝部署于主流服务器、NAS 及树莓派等 ARM 设备。
- **完全自动化**：通过 GitHub Actions 每日自动检查 Caddy 官方新版本，一旦发现更新，将自动编译并发布新的 Release。
- **与官方同步**：构建产物版本号与 Caddy 官方严格一致，确保您使用的是最新、最稳定的 Caddy 核心，并附带 SHA256 校验和。

---

### ⚡ 如何使用

只需三步，即可为您的域名启用基于 Cloudflare 的自动 HTTPS。

#### 第 1 步：获取 Cloudflare API Token

这是官方推荐的、权限更精细、更安全的方式。

1.  登录 Cloudflare 仪表板，前往 `“我的个人资料” -> “API 令牌”`。
2.  点击 `“创建令牌”`，然后选择使用 `“编辑区域 DNS”` 模板。
3.  在 `“区域资源”` 部分，将权限限制在需要管理证书的特定域名（区域）。
4.  创建并复制生成的 Token，它只会出现一次。

#### 第 2 步：选择一种配置方式

##### 选项 A：使用 Caddyfile (推荐)

Caddyfile 是最简单直观的方式。创建一个名为 `Caddyfile` 的文件：

```caddyfile
# 将 your.domain.com 替换为你的真实域名
your.domain.com, *.your.domain.com {
    # 配置 TLS，并指定使用 cloudflare DNS 插件
    tls {
        # Caddy 会自动从名为 CF_API_TOKEN 的环境变量中读取您的 API Token
        dns cloudflare {env.CF_API_TOKEN}
    }

    # 在这里配置你的服务，例如反向代理
    reverse_proxy localhost:8080
}
```

##### 选项 B：使用 JSON 文件 (高级)

对于 API 驱动或复杂场景，可以使用 JSON。创建一个名为 `config.json` 的文件：

```json
{
  "apps": {
    "http": {
      "servers": {
        "srv0": {
          "listen": [":443"],
          "routes": [
            {
              "handle": [{
                "handler": "reverse_proxy",
                "upstreams": [{ "dial": "localhost:8080" }]
              }]
            }
          ]
        }
      }
    },
    "tls": {
      "automation": {
        "policies": [{
          "subjects": ["your.domain.com", "*.your.domain.com"],
          "issuers": [{
            "module": "acme",
            "challenges": {
              "dns": {
                "provider": {
                  "name": "cloudflare",
                  "api_token": "{env.CF_API_TOKEN}"
                }
              }
            }
          }]
        }]
      }
    }
  }
}
```

#### 第 3 步：运行 Caddy

在启动 Caddy 前，先通过环境变量设置您的 API Token。

```bash
# 将 <Your_Cloudflare_API_Token> 替换为您复制的真实 Token
export CF_API_TOKEN="<Your_Cloudflare_API_Token>"

# 如果使用 Caddyfile，直接运行:
./caddy run

# 如果使用 JSON 文件，需指定配置文件路径:
./caddy run --config /path/to/your/config.json
```

---

### 🛠️ 工作原理

本仓库使用 GitHub Actions 自动化执行以下任务：

1.  **每日检查**：定时任务每日触发，访问 Caddy 官方仓库 API，获取最新的版本号。
2.  **版本比对**：检查本地仓库是否已存在该版本号的 Release。如果存在，则跳过本次任务。
3.  **交叉编译**：如果发现新版本，则启动构建任务，使用 `xcaddy` 为 `linux/amd64` 和 `linux/arm64` 两个平台交叉编译 Caddy。
4.  **插件验证**：在编译 `amd64` 版本后，会**严格执行验证步骤**，确保 Cloudflare 插件已成功集成。如果验证失败，构建将中止。
5.  **打包发布**：将编译好的二进制文件和 SHA256 校验和打包，并创建一个与官方版本号一致的 GitHub Release。

### 致谢

- [Caddy](https://github.com/caddyserver/caddy)
- [xcaddy](https://github.com/caddyserver/xcaddy)
- [caddy-dns/cloudflare](https://github.com/caddy-dns/cloudflare)