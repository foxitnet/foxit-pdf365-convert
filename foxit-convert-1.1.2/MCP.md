
## MCP 配置

### 前置条件

**必需 MCP Server**: `foxit-pdf365-mcp-server`

优先使用名为 `foxit-pdf365-mcp-server` 的 MCP server；若当前智能体暴露的是同一 PDF365 转换 MCP 的其它别名，以实际可用 server 名为准。

**MCP 配置**（VS Code `mcp.json` 示例，路径：`%APPDATA%\Code\User\mcp.json`）:

```json
{
  "servers": {
    "foxit-pdf365-mcp-server": {
      "type": "streamableHttp",
      "url": "https://open.pdf365.cn/mcp",
      "headers": {
        "X-API-KEY": "YOUR_API_KEY"
      }
    }
  },
  "inputs": []
}
```

**qclaw**（`openclaw.json` 示例）:

```json
{
  "mcp": {
    "servers": {
      "foxit-pdf365-mcp-server": {
        "url": "https://open.pdf365.cn/mcp",
        "transport": "streamable-http",
        "headers": {
          "X-API-KEY": "YOUR_API_KEY"
        }
      }
    }
  }
}
```

**WorkBuddy** 示例:

```json
{
  "mcpServers": {
    "foxit-pdf365-mcp-server": {
      "type": "http",
      "url": "https://open.pdf365.cn/mcp",
      "headers": {
        "X-API-KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

**Claude Code**（`~/.claude/settings.json` 示例）:

```json
{
  "mcpServers": {
    "foxit-pdf365-mcp-server": {
      "type": "url",
      "url": "https://open.pdf365.cn/mcp",
      "headers": {
        "X-API-KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

**Codex**（`~/.codex/config.json` 示例）:

```json
{
  "mcpServers": {
    "foxit-pdf365-mcp-server": {
      "url": "https://open.pdf365.cn/mcp",
      "headers": {
        "X-API-KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

**OpenCode**（`~/.opencode/config.json` 示例）:

```json
{
  "mcp": {
    "servers": {
      "foxit-pdf365-mcp-server": {
        "url": "https://open.pdf365.cn/mcp",
        "headers": {
          "X-API-KEY": "YOUR_API_KEY"
        }
      }
    }
  }
}
```

**Cursor**（`~/.cursor/mcp.json` 示例）:

```json
{
  "mcpServers": {
    "foxit-pdf365-mcp-server": {
      "type": "streamableHttp",
      "url": "https://open.pdf365.cn/mcp",
      "headers": {
        "X-API-KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

> 各客户端字段名略有差异，按各自格式配置相同字段。配置后重载 MCP 生效。
>
> ⚠️ `headers` 中的 `X-API-KEY` 值必须替换为用户自己的**完整** Key（从 https://www.pdf365.cn/skill_key 获取，本地存储于本技能目录下的 `skill_key` 文件，格式为 `PDF365-MCP-xxxxxxxx`）。自动配置 MCP 时，应读取本技能目录下 `skill_key` 文件内容**原样填入** `headers["X-API-KEY"]`，不得截断或去除 `PDF365-MCP-` 前缀。
>
> 🔧 自动配置写入 MCP 时读取本地 Key，同样须执行「读取 Key 时去除 BOM 与空白」的处理（用 `utf-8-sig` 解码并 `strip()`），避免 `\ufeff` 前缀或空白污染 MCP 配置中的 `X-API-KEY`。

> 🔒 **MCP 配置的运维安全约束（须与 [API_KEY.md](./API_KEY.md)「运维安全约束」一致）**：
> - **多客户端节制（最小暴露）**：每个客户端一份明文 `X-API-KEY` 就多一份泄漏面与轮换成本。自动配置时**仅为当前确实使用的客户端**写入；不得为未使用、已停用或已卸载的客户端批量复制 Key。
> - **安装包 / 分发排除**：任何安装、部署或分发用的 MCP 配置示例中，`X-API-KEY` 一律使用占位符 `YOUR_API_KEY`，**不得包含真实 Key**；分发 / 打包前将含真实 Key 的配置文件排除出包，防止真实 Key 被打进安装包 / 镜像 / 备份 / 同步。
> - **文件 ACL**：写入含 `X-API-KEY` 的 MCP 配置文件后，收紧其访问权限，仅允许当前用户读写（Unix 用 `chmod 600`，Windows 调整 ACL 移除其他用户 / 组读取权限）。
> - **日志脱敏**：自动配置或 Key 一致性校验过程中，任何打印 / 日志**不得输出完整 Key 或完整 `X-API-KEY` 请求头**，确需展示时仅显示脱敏形式（如 `PDF365-MCP-****xxxx`）；错误堆栈中同样**不得**包含明文 Key。
> - **命令历史保护**：curl 兜底一律以 `$API_KEY`（由 `skill_key` 注入环境变量）引用，**严禁**在命令行明文写 Key（避免写入 shell / PowerShell 历史）。
> - **轮换 / 撤销**：若配置中的 Key 疑似或确认泄漏、被分发或打包，按 [API_KEY.md](./API_KEY.md)「轮换 / 泄漏撤销」立即吊销并轮换，且**同步清理**所有已写入该 Key 的客户端 MCP 配置。

### MCP 调用模式

优先调用当前客户端已配置的 `foxit-pdf365-mcp-server` MCP 工具。若当前客户端没有配置 MCP server，**必须先尝试自动配置**，不得直接跳过到 curl 兜底。

#### 步骤 1：检查 MCP 是否已配置

检测当前客户端是否已注册 `foxit-pdf365-mcp-server` 的 MCP 工具（如 `createConvertTask`、`getConvertTaskStatus` 等工具是否可直接调用）。

- **已配置**：先执行 **Key 一致性校验**（读取本地 `skill_key` 与配置中的 `X-API-KEY` 比较，方法见「步骤 2→Key 一致性校验」）：
  - **一致** → 直接调用 MCP 工具，跳到「MCP 工具调用」章节。
  - **不一致** → 按「步骤 2→Key 一致性校验」的备份/更新/提示重载流程处理，完成后重载 MCP 再调用。
- **未配置**：进入步骤 2。

#### 步骤 2：自动配置 MCP（必须执行，不得跳过）

按当前运行环境检测对应的客户端配置文件，将 `foxit-pdf365-mcp-server` 配置写入：

| 客户端 | 配置文件路径 | 写入节点 | 配置内容 |
|---|---|---|---|
| **qclaw** | `~/.qclaw/openclaw.json` | `mcp.servers` | qclaw 配置示例 |
| **WorkBuddy** | `~/.workbuddy/mcp.json` | `mcpServers` | WorkBuddy 配置示例 |
| **Copilot / VS Code** | `%APPDATA%\Code\User\mcp.json` | `servers` | VS Code 配置示例 |
| **Claude Code** | `~/.claude/settings.json` | `mcpServers` | Claude Code 配置示例 |
| **Codex** | `~/.codex/config.json` | `mcpServers` | Codex 配置示例 |
| **OpenCode** | `~/.opencode/config.json` | `mcp.servers` | OpenCode 配置示例 |
| **Cursor** | `~/.cursor/mcp.json` | `mcpServers` | Cursor 配置示例 |

操作规则：
1. 读取配置文件（若不存在则创建）
2. 读取本地 `skill_key`（须执行「去除 BOM 与空白」处理，方法见上文「前置条件」），得到**期望 Key**
3. 检查是否已包含 `foxit-pdf365-mcp-server`：
   - **未包含** → 将对应配置写入目标节点（URL 用 `https://open.pdf365.cn/mcp`，`X-API-KEY` 用清洗后的本地完整 Key），保留文件中已有其他配置不变
   - **已包含** → 执行 **Key 一致性校验**（见下方「Key 一致性校验」）：仅当 server 名、URL、清洗后的 `X-API-KEY` **全部一致**时无需任何操作；任一不一致则执行「Key 一致性校验」的备份与更新
4. **仅当步骤 3 实际发生了写入或更新**时才提示用户重载 MCP 生效，等待用户确认重载完成后再继续调用 MCP 工具；未发生写入（如已存在且一致）则无需重载，直接继续调用 MCP 工具进入「MCP 工具调用」章节

> ⚠️ **Key 一致性校验（已含同名 server 时必须执行，不得仅凭同名即跳过）**：
> - 将配置中现有的 `X-API-KEY` 取出，**同样执行「去除 BOM 与空白」清洗**（`utf-8-sig` 解码 + `strip()`），再与本地 `skill_key`（清洗后）比较。
> - **一致**（server 名相同 + `url` 为 `https://open.pdf365.cn/mcp` + `X-API-KEY` 与本地清洗后 Key 相同）→ **无需写入、无需重载**（配置本就正确，强制重载只会打断调用），直接继续调用 MCP 工具，进入「MCP 工具调用」章节。
> - **不一致**（Key 已更换/过期、URL 不同、或 server 名存在但 Key 缺失）→ 按以下流程处理：
>   1. **更新**：将 `url` 更新为 `https://open.pdf365.cn/mcp`，将 `X-API-KEY` 更新为本地 `skill_key`（清洗后）的**完整** Key（含 `PDF365-MCP-` 前缀，不得截断），保留该 server 节点下其他字段（`type`、`transport` 等）不变
>   2. **仅当实际发生了写入/更新**时才提示用户重载/重启 MCP 使新 Key 生效，等待用户确认重载完成后再继续调用 MCP 工具；未发生写入则无需重载

> ⚠️ 自动配置是未配置时的**首选动作**，必须先尝试写入配置文件。只有当配置文件不存在且无法创建、或写入失败时，才进入步骤 3。

#### 步骤 3：curl 兜底（仅当自动配置失败时）

当步骤 2 中所有客户端配置文件均不存在或写入失败时，使用 `curl` 直接调用 MCP HTTP 接口完成转换流程。

**初始化会话**（首次调用前必须执行，获取 `Mcp-Session-Id`）：

```bash
curl -s -D - \
  "https://open.pdf365.cn/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-KEY: ${API_KEY}" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"copilot","version":"1.0"}},"id":1}'
```

从响应头中提取 `Mcp-Session-Id`，后续所有请求须携带该头。

**发送初始化通知**（initialize 之后、tools/call 之前）：

```bash
curl -s -N "https://open.pdf365.cn/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-KEY: ${API_KEY}" \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
```

**查看工具列表和 schema**：

```bash
curl -s -N "https://open.pdf365.cn/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-KEY: ${API_KEY}" \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":1}'
```

**调用工具**：

```bash
curl -s -N "https://open.pdf365.cn/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-KEY: ${API_KEY}" \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"TOOL_NAME","arguments":{}},"id":1}'
```

将 `TOOL_NAME` 替换为实际工具名（如 `createConvertTask`、`getConvertTaskStatus`、`downloadConvertResult`），将 `arguments` 替换为工具 schema 要求的参数。从 `result.content[0].text`、`result.structuredContent` 或 SSE `data:` 事件中解析返回结果。

> ⚠️ 首次调用某 MCP 工具前，或不确定参数时，先读取对应工具 descriptor/schema；本文参数只是快速参考，实际以 schema 为准。