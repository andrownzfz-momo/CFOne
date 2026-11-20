# CFOne · 单文件 Cloudflare 代理面板

> 融合 CFPilot / CFAPEX / cfnew / Cloudflare-vless-trojan 四个上游项目的设计，
> 目标：**最方便部署、最稳定、安全性最高**。

部署方式与 [cfnew](https://github.com/byjoey/cfnew) 一致：**下载产物 → 控制台粘贴 → 设变量 → 部署**，全程点鼠标，不需要任何工具或命令。

## 特性

| 维度 | 实现 |
| --- | --- |
| **部署** | 单文件双形态：Worker 粘贴 / Pages Advanced Mode，必填仅 `A` 一个变量 |
| **稳定** | 无关请求零 KV 读取、10 秒窗口合并缓存、配置版本号失效；出站候选竞速 + 假活检测（无首字节即换路）；WS close reason 123 字节截断；连接/流量上限保护 |
| **安全** | PBKDF2 口令散列 + HttpOnly 会话 + 常量时间比较 + IP 限速；订阅签名令牌与面板密码分离；SSRF 防护（拒绝内网/回环/保留段）；构建时静态脱敏（中文/品牌名零明文残留） |
| **订阅** | UA 自动识别 + 显式 `?format=`：**Clash / Stash / sing-box / Surge / Loon / QuantumultX / XHTTP / Base64 / 明文** 共 9 种 + 家宽链式入口；Clash·Mihomo·Verge·Karing→YAML · Stash→YAML · Surge→INI · Loon→INI · Quantumult→QX · sing-box→JSON · v2rayN·Shadowrocket·NekoBox→Base64 · 浏览器→明文 |
| **协议** | VLESS/WS、Trojan/WS、**VLESS-XHTTP（stream-one）**；XHTTP 与 WS 共用路径，靠 method 区分（POST→XHTTP、Upgrade→WS） |
| **出站** | 直连 / ProxyIP 竞速、SOCKS5 / HTTP / HTTPS 代理、出站方式三选一（竞速/直连优先/只走代理）；Path 级 `pyip` / `s` / `qj` 单节点覆盖 |
| **面板** | 图形化管理：UUID、WS 路径、Trojan 开关、ProxyIP、出站代理、优选 IP 列表（KV 持久化）、伪装页 |
| **保护** | **Free CPU 预算保护**（`freeCpu=yes`）：每连接双向合计 ≤384 KiB 且 ≤96 块，超限断开，防止免费账号 CPU 超额 |
| **家宽** | **家宽链式**（`jk=yes` + `?format=jk/vg/jiakuan`）：CF 入口经 `dialer-proxy` 接家宽出口（客户端→家宽→CF→目标），内置含**美国**多出口池，可用 `JK_NODE` 覆盖 |

## 部署（手动，控制台点鼠标）

> 下载 `dist/CFOne-worker.js` 的内容（或 `dist/CFOne-pages.zip` 整包上传），照下面 8 步走。全程 Cloudflare 控制台，不装任何工具。

**步骤 1：创建 KV 命名空间**

左侧菜单 **Workers 和 Pages** → 顶部 **KV** 标签 → **Create namespace** → 名字随意（如 `cfone-kv`）→ 创建。
记下返回的 **Namespace ID**（一串 32 位十六进制）。

**步骤 2：新建 Worker**

左侧 **Workers 和 Pages** → **Create** → **Worker** → 起名（如 `cfone`）→ **Deploy**。

**步骤 3：粘贴代码**

点进刚建的 Worker → **Edit code**（在线编辑器）→ 全选删掉默认内容 → 粘贴 `dist/CFOne-worker.js` 的**全部内容** → **Save and deploy**。

**步骤 4：设置变量 `A`**

Worker 详情 → **Settings** → **Variables and Secrets** → **Add**：

| 字段 | 填什么 |
| --- | --- |
| Variable name | `A` |
| Value | 你的面板密码（长随机串，如 `ded98a79260487ac6c8b3d32`） |
| Type | `Text` |

→ **Save**。

**步骤 5：绑定 KV**

同一页面 **Settings** → **Bindings** → **Add** → **KV namespace**：

| 字段 | 填什么 |
| --- | --- |
| Variable name | `K`（必须是大写 K） |
| KV namespace | 选步骤 1 创建的命名空间 |

→ **Save** → 重新 **Deploy** 一次。

**步骤 6：确认兼容性日期**

**Settings** → **Version** / **Compatibility date** → 设为 `2026-01-20`。

**步骤 7：拿到访问地址（关键，很多人卡这）**

**Cloudflare 的完整地址必须带账号子域**：

```
https://<Worker名>.<账号子域>.workers.dev
```

- 你的 Worker 名：`cfone`
- 你的账号子域：**Workers 和 Pages** 首页顶部能看到（如 `andrownzfz`）
- 完整地址：`https://cfone.andrownzfz.workers.dev`

> ⚠️ 少了中间那段 `<账号子域>.` 的短地址 `https://cfone.workers.dev` **DNS 不存在，打不开**。

**步骤 8：访问面板（根路径直达）**

```
https://<Worker名>.<账号子域>.workers.dev/
```

例：`https://cfone.andrownzfz.workers.dev/` → 直接显示面板登录页 → 输入 `A` 的值登录。

> 只记一个密码即可，不用记 `/密码` 路径。旧的 `/<A的值>` 路径仍可用（向后兼容）。

### 访问不通排查：`error code: 1042`

**症状**：地址能解析、不是超时，但返回 `HTTP 404` + `error code: 1042`。换脚本、重部署都没用。

**根因**：账号的 **workers.dev 通配路由未启用**（API 里 `wildcard_host: false`）。边缘收到请求，但拒绝转给任何 Worker，于是吐 1042。

**这不是脚本 / KV / 权限的问题** —— 部署本身是成功的，只是「访问通道」没开。

**判断方法**（任一条命中即为该问题）：

1. 部署返回成功（脚本列表里能看到、bindings 正确），但访问任何路径都 1042
2. 用一个只 `return new Response("ok")` 的最小脚本测试，**也是 1042** → 证明与脚本无关
3. API 查 `GET /accounts/{id}/workers/settings` 返回 `wildcard_host: false`

**解决（控制台手动开，1 分钟）**：

1. 登录 https://dash.cloudflare.com
2. 左侧 **Workers 和 Pages** → 首页顶部/右上找 **Workers.dev 子域** 设置（⚙️ 或 Settings）
3. 打开开关：**Enable workers.dev**（子域值应与你地址中间那段一致，如 `andrownzfz`）
4. **等 1~2 分钟**让路由生效
5. 用完整地址重访

> 该开关 **API 不开放**（PUT/PATCH 全部 405/409），只能控制台点。

**备用方案：绑定自定义域名（不依赖 workers.dev，一定通）**

若找不到开关或开了仍 1042，用你账号下已接入的域名：

1. Worker 详情 → **Settings** → **Domains & Routes** → **Add Custom Domain**
2. 输入子域，如 `cf.你的域名.com`（需该域名已接入 Cloudflare）
3. 按提示在 **DNS** 加一条 CNAME，目标指向该 Worker（控制台会给出确切值）
4. 等 DNS 生效 + 证书签发（几分钟）
5. 访问 `https://cf.你的域名.com/` 直接进面板

**其他常见访问问题**：

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 打开是「Welcome to Cloudflare」欢迎页 | 未设密码 `A`，或设了 `FAKE=yes` | 设好 `A` 并重新部署；`FAKE=yes` 会把根路径改回伪装页 |
| 一直转圈 / 超时 | `*.workers.dev` 被本地网络阻断 | 开代理，或改用上面的自定义域名方案 |
| HTTP 000 连不上 | 地址写成短子域（少了 `.账号子域.`） | 用完整地址 `https://<名>.<账号子域>.workers.dev` |
| 403 / 登录失败 | 密码错或会话过期 | 重新输入 `A` 的值登录 |

## 使用

1. 浏览器直接打开 `https://<部署地址>/`（**根路径即面板**），输入面板密码登录
2. 在「优选节点」页填写优选 IP（或留空用内置池）
3. 在「设置」页配置 UUID、WS 路径、Trojan、出站等
4. 在「概览」页复制订阅地址，导入客户端

> **访问方式**：默认根路径 `/` 直接显示面板登录页，只记一个密码即可。旧的 `/<密码>` 路径仍可用（向后兼容）。若要恢复「根路径伪装、面板藏在 `/<密码>`」的旧行为，设环境变量 `FAKE=yes` 后重新部署——此时根路径返回伪装页，扫描器看到的是假页面。未设密码（`A` 为空）时根路径始终为伪装页。

### 订阅格式（9 种 + 家宽链式入口）

订阅地址默认按客户端 UA 自动适配；也可显式加 `?format=` 指定：

| `format=` | 输出 | 适配客户端 |
| --- | --- | --- |
| `clash` | Clash YAML | Clash / Mihomo / Verge / Karing |
| `stash` | Clash 兼容 YAML | Stash |
| `surge` | Surge INI | Surge |
| `loon` | Loon INI | Loon |
| `qx` | QuantumultX 配置 | Quantumult X |
| `singbox` | sing-box JSON | sing-box |
| `xhttp` | VLESS-XHTTP 链接 | 支持 XHTTP 的客户端 |
| `base64` | Base64 链接集 | v2rayN / Shadowrocket / NekoBox |
| `plain` | 明文链接集 | 浏览器 / 通用 |
| `vg` / `jk` / `jiakuan` | **家宽链式 Clash YAML**（需 `jk=yes`） | 支持链式的 Clash / Mihomo |

例：`https://<部署地址>/s/<订阅令牌>?format=surge`

**XHTTP**：`?format=xhttp` 返回 `type=xhttp&mode=stream-one` 链接。服务端监听同一隧道路径的 **POST** 请求（与 WS 的 `Upgrade` 复用同一 path，靠 method 区分），出站复用竞速/假活/代理/SSRF 全套防护。

**家宽链式**：先设 `jk=yes`（环境变量 / 面板），再访问 `?format=jk`。输出 CF 入口 × 每个家宽出口的全组合链式节点（`dialer-proxy`，客户端→家宽→CF→目标）。默认内置含**美国**的多出口池（占位）；真实节点用 `JK_NODE` 覆盖（JSON 数组）：

```json
[{"name":"US-LA-01","type":"ss","server":"1.2.3.4","port":443,"cipher":"aes-256-gcm","password":"xxx"}]
```

**Free CPU 保护**：`freeCpu=yes` 后，每条连接双向合计 ≤384 KiB 且 ≤96 个数据块，超限即断开（客户端自行重连），防止免费账号 CPU 超额。对新连接生效，不支持 path 覆盖。

### Path 级覆盖（单节点）

在 VLESS/Trojan 分享链接的 `path` 查询参数中可覆盖全局配置：

```
/?ed=2048&pyip=1.2.3.4
/?ed=2048&pyip=proxy.example.com:443
/?ed=2048&s=socks5://user:pass@host:1080
/?ed=2048&qj=only
```

优先级：path 参数 > KV 配置 > 环境变量 > 默认值。

## 安全说明

- **面板密码 `A` 即访问凭证**：持有者可查看订阅、修改配置。请使用长随机值，不要公开。
- **订阅令牌与面板密码分离**：订阅地址 `/s/<token>` 的 token 是 `HMAC(A, "sub")` 派生，与面板路径不同，泄露订阅地址不会暴露面板。
- **SSRF 防护**：出站目标经过校验，拒绝内网、回环、链路本地、保留段地址。
- **构建脱敏**：产物中中文、品牌名零明文残留，字符串经 base64 解码表运行时还原。
- **会话**：登录后下发 HttpOnly + SameSite=Strict Cookie，服务端内存记录 nonce，常量时间比较校验。

## 构建与测试

```bash
node scripts/build.mjs        # 构建 dist/（脱敏）
node scripts/smoke-test.mjs   # 冒烟测试（31 项）
```

> 手动部署**不需要**构建——直接用仓库里的 `dist/` 产物即可。改了 `src/worker.js` 才需要重新 `build`。

## 目录结构

```
CFOne/
├── src/worker.js            # 源码（可读，__SFX 标记待脱敏字符串）
├── scripts/
│   ├── build.mjs            # 脱敏构建 → dist/
│   └── smoke-test.mjs       # 冒烟测试（31 项）
└── dist/
    ├── CFOne-worker.js      # Workers 粘贴 / Pages 根文件
    ├── _worker.js           # Pages Advanced Mode 根文件
    └── CFOne-pages.zip      # Pages 上传包
```

## 致谢

融合自以下上游项目的设计：
- [xmwengxing/CFPilot](https://github.com/xmwengxing/CFPilot) — 双项目拆分、真连接优选、PBKDF2 鉴权、静态脱敏
- [shitouxiaoge-commits/CFAPEX](https://github.com/shitouxiaoge-commits/CFAPEX) — 单文件双形态、KV 读量优化、运行时解码表
- [byjoey/cfnew](https://github.com/byjoey/cfnew) — UA 自动识别多格式订阅、出站代理三选一、手动部署流程
- [yonggekkk/Cloudflare-vless-trojan](https://github.com/yonggekkk/Cloudflare-vless-trojan) — Path 级 proxyip 覆盖、13 端口优选池

## License

AGPL-3.0
