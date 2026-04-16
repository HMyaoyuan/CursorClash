# CursorClash

在中国大陆使用 Cursor 时，通过 Clash 实现**仅 Cursor 走美国节点**的分流方案。解决使用特定模型（如 Claude Opus 4.6）时必须开 TUN 增强模式，但开了之后微信、WPS、腾讯文档等国内应用无法正常使用的问题。

## 问题背景

| 场景 | 现象 |
|------|------|
| 不开 TUN，用全局代理 | Cursor 无法使用 Opus 4.6（DNS 污染/SNI 检测） |
| 开 TUN + 全局美国节点 | Cursor 正常，但微信/WPS/腾讯文档全部异常 |
| **本方案：开 TUN + 规则模式** | **Cursor 走美国节点，其他应用走原有规则，全部正常** |

## 原理

Clash 的 TUN（增强模式）在网络层接管所有流量（包括 DNS 解析），然后由 `rules` 规则决定每条连接走哪个节点。通过 `PROCESS-NAME` 规则，可以按进程名精确分流：

```
所有流量 → TUN 接管 → rules 逐条匹配
  ├── PROCESS-NAME 命中 Cursor 进程 → 美国节点
  └── 未命中 → 继续走原有规则（国内直连、其他代理等不受影响）
```

关键点：
- TUN 解决了 DNS 污染问题（DNS 查询也走代理隧道）
- `PROCESS-NAME` 实现了按应用分流，不影响其他软件
- 选择「规则」模式而非「全局」模式，让原有的域名/IP 分流规则继续生效

## 环境要求

- macOS 系统
- Clash 客户端支持 TUN 和 `PROCESS-NAME` 规则：
  - **ClashX Pro**（已验证可用）
  - Clash Verge Rev（推荐，基于 mihomo 内核，支持更好）
  - Stash
- 机场订阅中有美国节点

## 完整配置步骤

### 第 1 步：确认 Cursor 进程名

打开 Cursor，在终端运行：

```bash
ps aux | grep -i "[C]ursor"
```

你会看到类似这样的进程：

```
/Applications/Cursor.app/Contents/MacOS/Cursor
/Applications/Cursor.app/Contents/Frameworks/Cursor Helper.app/.../Cursor Helper
/Applications/Cursor.app/Contents/Frameworks/Cursor Helper (Renderer).app/.../Cursor Helper (Renderer)
/Applications/Cursor.app/Contents/Frameworks/Cursor Helper (Plugin).app/.../Cursor Helper (Plugin)
/Applications/Cursor.app/Contents/Frameworks/Cursor Helper (GPU).app/.../Cursor Helper (GPU)
```

进程名就是路径最后一段。一般这 5 个就够了：

| 进程名 | 作用 |
|--------|------|
| `Cursor` | 主进程 |
| `Cursor Helper` | 网络/工具进程（最关键，AI 请求走这里） |
| `Cursor Helper (Renderer)` | 界面渲染 |
| `Cursor Helper (Plugin)` | 插件/扩展 |
| `Cursor Helper (GPU)` | GPU 加速 |

### 第 2 步：打开 Clash 配置文件

**ClashX Pro**：菜单栏点击 Config → Open config folder，打开你正在使用的 `.yaml` 文件。

### 第 3 步：确认美国节点名称

在配置文件的 `proxies:` 段中找到美国节点，记下完整名称。例如：

```yaml
- {name: 🇺🇲 美国A01 | IEPL | x1.5, server: ..., type: ss, ...}
- {name: 🇺🇲 美国A02 | IEPL | x1.5, server: ..., type: ss, ...}
```

### 第 4 步：新增 Cursor 专用策略组

在 `proxy-groups:` 的末尾添加一个新的策略组：

```yaml
proxy-groups:
  # ... 你原有的策略组保持不变 ...

  - name: 💻 Cursor 代理
    type: select
    proxies:
      - 🇺🇲 美国A01 | IEPL | x1.5    # 替换成你的美国节点名
      - 🇺🇲 美国A02 | IEPL | x1.5    # 替换成你的美国节点名
      - 🔰 选择节点                     # 你的主策略组，作为备选
      - DIRECT                          # 不走代理时选这个
```

这样做的好处是可以在 ClashX Pro 界面里随时切换 Cursor 走哪个节点，不用改配置文件。

### 第 5 步：在 rules 最前面插入进程规则

在 `rules:` 列表的**最前面**添加以下 5 条规则（规则从上往下匹配，靠前的优先级高）：

```yaml
rules:
  - PROCESS-NAME,Cursor,💻 Cursor 代理
  - PROCESS-NAME,Cursor Helper,💻 Cursor 代理
  - PROCESS-NAME,Cursor Helper (Renderer),💻 Cursor 代理
  - PROCESS-NAME,Cursor Helper (Plugin),💻 Cursor 代理
  - PROCESS-NAME,Cursor Helper (GPU),💻 Cursor 代理
  # ↓↓↓ 你原有的所有规则保持不变 ↓↓↓
  - DOMAIN-SUFFIX,google.com,Proxy
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
  # ... 等等
```

### 第 6 步：重载配置

ClashX Pro 菜单栏 → Config → **Reload config**。

### 第 7 步：切换到正确的模式

这一步非常关键：

| 设置项 | 选择 |
|--------|------|
| 模式 | **规则（Rule）** —— 不要选全局！ |
| 增强模式（TUN） | **开启** |

以前你可能用的是「全局 + 美国节点」，现在改成**「规则 + TUN」**。Cursor 会被 PROCESS-NAME 规则自动路由到美国节点。

## 日常使用

配置完成后，日常操作只需要：

1. 打开 ClashX Pro
2. 确认增强模式（TUN）开启
3. 确认模式为「规则」
4. 正常使用一切应用

不需要来回切换节点或模式。Cursor 自动走美国，微信/WPS/腾讯文档自动走直连或原有规则。

**如果临时不需要 Cursor 走美国节点**：在 ClashX Pro 界面把「💻 Cursor 代理」策略组切换到 `DIRECT` 即可。

## 验证方法

### 方法一：Dashboard 查看连接

打开 ClashX Pro 的 Dashboard → Connections，搜索 `Cursor`，确认连接走的是 `💻 Cursor 代理` → 美国节点。

### 方法二：实际测试

- 在 Cursor 中使用 Opus 4.6 对话，确认正常响应
- 打开微信发消息，确认正常
- 打开 WPS / 腾讯文档，确认正常打开和编辑

## 故障排查

### PROCESS-NAME 规则不生效

- 升级 ClashX Pro 到最新版本
- 如果仍不行，考虑迁移到 [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev)（基于 mihomo 内核，`PROCESS-NAME` 支持更完善）

### Cursor 仍然无法连接

1. 在 Dashboard → Connections 中搜索 Cursor 相关连接
2. 确认是否匹配到了 `💻 Cursor 代理` 规则
3. 如果没匹配到，用 `ps aux | grep -i cursor` 重新确认进程名
4. 检查规则是否确实放在了 `rules:` 的最前面

### DNS 相关问题

确保配置文件中有正确的 DNS 设置：

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip  # 或 redir-host
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
```

### 微信/WPS 仍然异常

这些应用应该走原有规则（通常是 `GEOIP,CN,DIRECT`）。如果异常：
1. 在 Dashboard 中搜索 WeChat / wpsoffice，确认它们走的是 DIRECT
2. 如果它们意外匹配到了代理规则，可以在 Cursor 规则后面、原有规则前面，手动添加直连规则：
   ```yaml
   - PROCESS-NAME,WeChat,DIRECT
   - PROCESS-NAME,wpsoffice,DIRECT
   ```

## 实际配置示例

以下是本项目作者实际使用的配置片段（来自 iKuuu 机场订阅）：

**策略组部分**（添加在 proxy-groups 末尾）：

```yaml
  - name: 💻 Cursor 代理
    type: select
    proxies:
      - 🇺🇲 美国A01 | IEPL | x1.5
      - 🇺🇲 美国A02 | IEPL | x1.5
      - 🔰 选择节点
      - DIRECT
```

**规则部分**（添加在 rules 最前面）：

```yaml
rules:
 - PROCESS-NAME,Cursor,💻 Cursor 代理
 - PROCESS-NAME,Cursor Helper,💻 Cursor 代理
 - PROCESS-NAME,Cursor Helper (Renderer),💻 Cursor 代理
 - PROCESS-NAME,Cursor Helper (Plugin),💻 Cursor 代理
 - PROCESS-NAME,Cursor Helper (GPU),💻 Cursor 代理
 # 后面是原有规则...
```

## 适用于其他应用

同样的方法可以用于任何需要走特定节点的应用。只需要：
1. 用 `ps aux | grep -i <应用名>` 找到进程名
2. 添加对应的 `PROCESS-NAME` 规则和策略组

例如给 ChatGPT 桌面客户端也配一个美国节点：

```yaml
  - PROCESS-NAME,ChatGPT,💻 Cursor 代理
```

## License

MIT
