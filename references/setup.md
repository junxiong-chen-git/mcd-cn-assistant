# 麦当劳 MCP 接入(一次性)

服务器:`https://mcp.mcd.cn`,协议 Streamable HTTP,认证头 `Authorization: Bearer <MCP_TOKEN>`。

## 1. 申请 Token(用户自己操作)

1. 打开 https://open.mcd.cn/mcp,右上角【登录】,用手机号验证码登录。
2. 点右上角【控制台】→【激活】申请 MCP Token → 同意服务协议。
3. 复制 Token。Token 与手机号绑定,妥善保管,不要贴进聊天/日志。

## 2. 写入 OpenClaw 配置

**推荐:token 只进网关环境变量,配置里用占位符。** 配置加载时会按网关进程环境替换 `${MCD_MCP_TOKEN}`。

1. 由用户自行把变量加到网关服务环境(不要经过 chat,也不要把值发给我)。网关一般由 systemd user 服务 `openclaw-gateway.service` 拉起:
   - `systemctl --user edit openclaw-gateway.service`,加 `[Service]` 段与 `Environment=MCD_MCP_TOKEN=<token>`;
   - 或用权限 0600 的 EnvironmentFile。
2. 写配置(单引号防止 shell 展开):

```bash
openclaw mcp set mcd-mcp '{"url":"https://mcp.mcd.cn","transport":"streamable-http","headers":{"Authorization":"Bearer ${MCD_MCP_TOKEN}"}}'
```

3. 重启网关,让环境变量与配置生效。**改配置/重启服务需先经用户授权**(AGENTS.md 强制授权规则)。

**备选(仅个人机,会落盘明文):**

```bash
openclaw mcp add mcd-mcp --url https://mcp.mcd.cn --transport streamable-http --header 'Authorization=Bearer <token>'
```

明文 token 会写进 `~/.openclaw/openclaw.json`(0600)。这条命令含明文,不要贴到聊天或日志。

## 3. 验证

- `openclaw mcp doctor mcd-mcp`:静态检查(会标记明文敏感请求头)。
- `openclaw mcp probe mcd-mcp --json`:真实连接并列工具;看到 `campaign-calendar`、`calculate-price`、`create-order` 等即接入成功。
- `openclaw mcp reload`:让当前进程用上新配置(网关进程可能需重启)。

## 4. 排错

- **401**:token 无效/过期/没带上 → 确认 `Authorization` 头存在,且 `${MCD_MCP_TOKEN}` 真的被替换(网关环境里确实有该变量);不要把 `${MCD_MCP_TOKEN}` 字面量发出去。
- **429**:超过 600 次/分钟 → 降频,减少比价组合数。
- **工具列表为空**:确认 transport = `streamable-http`、URL = `https://mcp.mcd.cn`(不带路径)、请求头格式为 `Bearer <token>`。
