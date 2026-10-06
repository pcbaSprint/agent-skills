# pcbaSprint Agent Skills

**让 AI 助手帮你下单做电路板。**

[晚成鸟 pcbaSprint](https://www.pcbasprint.com) 是面向 PCB/SMT 行业的智能制造平台，支持从 Gerber 文件上传到成品发货的全流程在线管理。本仓库提供 AI Agent 技能（Skills），让你的 AI 助手直接对接 pcbaSprint 平台，用自然语言完成下单、查单、支付等操作。

## 你能对 AI 说什么

```
"帮我做一个 5 片的双面板，FR4 板材，沉金工艺，加急 48 小时"
"上传这个 Gerber 文件，BOM 也在里面，帮我下个 SMT 贴片的单"
"查一下我最近的订单到哪一步了"
"上次那个单再返 10 片"
"帮我看看有什么优惠券可以用"
```

AI 会自动完成文件上传、工艺参数配置、地址匹配、订单创建——你只需要说需求。

## 快速开始

### 1. 安装技能

```bash
npx skills add pcbaSprint/agent-skills
```

### 2. 连接 MCP 服务

在你的 AI Agent 平台中添加 MCP 服务器：

```
名称: pcb-customer
类型: streamable-http
地址: https://mcp.pcbasprint.com/mcp
```

### 3. 登录

首次使用时，AI 会引导你完成 OAuth 登录（浏览器跳转），登录后即可使用全部功能。

## 支持的平台

| 平台 | 安装方式 |
|------|---------|
| **Claude Code** | `npx skills add pcbaSprint/agent-skills` |
| **Qoder** | 复制到 `~/.qoder/skills/` 或项目 `.qoder/skills/` |
| **Cursor** | 复制到 `.cursor/skills/` |
| **Codex** | 将 SKILL.md 内容添加到项目 instructions |
| **WorkBuddy** | 添加到自定义指令 |
| **其他 Agent** | 将 SKILL.md 内容粘贴到系统提示词或 rules 文件 |

> 所有支持 [MCP 协议](https://modelcontextprotocol.io) 的 Agent 平台均可对接 pcbaSprint 后端服务。

## 技能详情

### customer-place-order — 客户端自助下单

覆盖 PCB/PCBA 下单全流程，与网页端体验一致：

| 能力 | 说明 |
|------|------|
| 文件上传 | Gerber/BOM/坐标文件自动解析分类，支持 zip 压缩包和 PCB 设计源文件 |
| PCB 制板 | 层数、板材、板厚、铜厚、表面处理、阻焊颜色、过孔处理、阻抗控制等全参数 |
| SMT 贴片 | 数量、贴装面、元器件来源（邮寄/代购/部分代购）、客供补差 |
| DIP 插件 | 数量、插件方式（手工/波峰焊）、元器件来源 |
| 其他服务 | 钢网、三防漆涂覆、组装测试 |
| 收货地址 | 地址管理、默认地址、手动填写 |
| 优惠券 | 领券中心、可用券查询、下单抵扣 |
| 支付 | 微信支付、支付宝，支付状态查询 |
| 开票 | 开票资料管理、开票申请（普票/专票） |
| 返单 | 按历史订单快速返单，沿用文件和工艺配置 |
| 草稿 | 保存/恢复下单草稿 |

## 关于晚成鸟

[晚成鸟](https://www.pcbasprint.com) 专注于 PCB 打样和 SMT 贴片的小批量智能制造，提供：

- **在线计价** — 上传 Gerber 即时报价，透明定价
- **智能 BOM 匹配** — AI 自动识别物料，减少人工核对
- **全流程可视** — 从下单到发货，每个节点实时追踪
- **加急交付** — 最快 24 小时出货

## License

Apache 2.0
