# pcbaSprint Agent Skills

PCB/SMT 智能制造 AI Agent 技能集，兼容 skills.sh 规范。

## 安装

```bash
npx skills add pcbaSprint/agent-skills
```

## 技能列表

| 技能 | 说明 | 依赖 |
|------|------|------|
| [customer-place-order](./customer-place-order/) | 客户端自助下单（PCB/SMT/DIP 全流程） | [pcb-customer MCP](https://mcp.pcbasprint.com/mcp) |

## 手动安装

不使用 `npx skills add` 的平台，可直接将技能文件夹复制到 Agent 的 skills 目录：

- **Qoder**: `~/.qoder/skills/` 或项目 `.qoder/skills/`
- **Claude Code**: 项目根目录或 `~/.claude/skills/`
- **Cursor**: `.cursor/skills/`
- **其他**: 将 SKILL.md 内容粘贴到系统指令 / rules 文件

## MCP 服务

本技能需要连接 pcb-customer MCP 服务：

| MCP 服务 | 地址 | 用途 |
|----------|------|------|
| pcb-customer | `https://mcp.pcbasprint.com/mcp` | 客户端下单、支付、开票 |

## License

Apache 2.0
