---
name: customer-place-order
description: 客户端自助下单，支持 PCB 制板/SMT 贴片/DIP 插件全流程。上传 Gerber/BOM 文件，配置工艺参数，创建订单并支付。当用户说"下单"、"客户端下单"、"PCB下单"、"打样"、"SMT下单"、"创建订单"、"返单"、"customer下单"时触发。
roles: [customer]
owner: da
requires_approval: true
---

# 客户端自助下单

通过 `pcb-customer` MCP 帮助客户创建 PCB/PCBA 订单。覆盖文件上传、工艺配置、
地址管理、优惠券、支付、开票全流程，与网页端（pcb-customer-web）下单体验一致。

## 平台接入

后端已部署 MCP 服务，支持 MCP 的平台添加以下服务器配置即可调用全部操作：

```
名称: pcb-customer
类型: streamable-http
地址: https://mcp.pcbasprint.com/mcp
```

> 首次使用需完成 OAuth 授权登录（浏览器跳转），登录后工具自动可用。

## 前置约束

- 用户必须已完成 OAuth 认证（未认证时调用任何工具会返回 401，引导用户完成登录）
- **产品名**（`product_name`）必填——有文件时从文件名自动提取，无文件时必须向用户询问
- **至少启用一项服务**（PCB/SMT/DIP/钢网/三防漆/组装测试 至少开一个）
- 文件上传后后端自动解析分类（Gerber/BOM/坐标），无需手动指定文件类型

## 下单流程

### Step 0：确定下单模式

三种入口，先问清楚用户意图：

| 用户意图 | 流程 |
|----------|------|
| 新建订单 | Steps 1-7 完整流程 |
| 返单 | Step 8 返单快捷流程 |
| 继续草稿 | Step 9 草稿恢复流程 |

### Step 1：收货地址

**操作**: `list_addresses`

- 有默认地址 → 展示给用户确认："将寄送到 XXX，电话 XXX，可以吗？"
- 用户想换新地址 → `create_address(receiver_name, receiver_phone, province, city, district, detail_address)`
- 用户想手动填写（不保存） → 直接在 Step 5 传手动地址字段

**手机号格式**: 11 位手机号 `1xxxxxxxxxx`

### Step 2：文件上传（可选）

客户下单通常有 Gerber/BOM 文件。上传后后端自动解析，返回文件分类结果和 PCB 参数。

#### 2a. 本地文件上传（统一路径）

所有本地文件统一使用 `upload_order_file` 上传。该工具通过 multipart/form-data 直传后端，
不经过 base64 编码，大小文件均适用（上限 100MB）。

**上传步骤**：

1. **准备文件**——未压缩的一律先打 zip：

   | 用户给的是 | 处理 |
   |-----------|------|
   | 文件夹（Gerber/坐标/BOM 混在一起） | `zip -r 资料名.zip <文件夹路径>` |
   | 单个 PCB 设计源文件（.kicad_pcb/.PcbDoc/.brd/.asc 等） | `zip 资料名.zip xxx.kicad_pcb` |
   | 已是 .zip / .rar | 直接用 |
   | 单个 .xlsx/.xls/.csv/.txt BOM 表（无 Gerber） | 也打 zip（后端只接受压缩包） |

2. **获取文件字节数**（用于判断上传方式）：
   ```bash
   stat -f%z file.zip    # macOS
   stat -c%s file.zip    # Linux
   ```

3. **上传**——按文件大小选择方式：

   **常规文件（≤10MB）**：直接调用 `upload_order_file(filename, file_base64=<base64>)`

   **大文件（>10MB）**：使用分块上传三件套，避免单次传输超时：
   1. `start_chunked_upload(filename, total_size)` → 获取 `uploadId`
   2. 按 ~256KB 分片，逐片调用 `upload_file_chunk(upload_id, index, chunk_base64)`
   3. 全部分片传完后调用 `complete_chunked_upload(upload_id)`

> **为什么不用 OSS 直传**：客户端 MCP 未暴露 OSS STS 凭证工具，`upload_order_file` 已
> 通过 multipart 直传后端，与管理端网页端上传效果一致，无需额外 OSS 步骤。

#### 2b. 公网 URL 文件

直接调用 `upload_order_file(filename, file_url="https://...")`

> 注意：MCP 服务运行在云端，无法访问内网 URL（192.168.x / 10.x / 172.16-31.x / 127.x）。
> 内网文件请用 `upload_order_file(file_base64=...)` 或分块上传。

#### 2c. 上传返回值（重要）

上传成功后返回以下字段，**全部传递给 `create_order`**：

| 返回字段 | 传给 create_order 的参数 |
|----------|--------------------------|
| `fileId` | `file_id` |
| `fileKey` | `file_key` |
| `gerberFileId` | `gerber_file_id` |
| `gerberFilePath` | `gerber_file_path` |
| `bomFileId` | `bom_file_id` |
| `bomFilePath` | `bom_file_path` |
| `coordinateFileId` | `coordinate_file_id` |
| `coordinateFilePath` | `coordinate_file_path` |
| `pcbFileId` | `pcb_file_id` |
| `pcbFilePath` | `pcb_file_path` |

同时返回 `pcbParameters`（板宽/高/层数）、`detectedMountingSide` 等，用于自动填充默认值。

#### 2d. 文件命名建议

zip 内文件按常规命名以便后端正确分类：
- Gerber: `.gbr`/`.gbl`/`.gtl`/`.gbo` 等
- BOM: `BOM.xlsx`/`BOM.csv`
- 坐标: `pick-place.csv`/`坐标.csv`（文件名不要含"坐标"以外的中文关键词，避免干扰分类）

### Step 3：收集下单参数

按以下顺序引导用户提供信息。**有默认值的参数不必主动询问**，仅在用户想定制时才问。

#### 3a. 产品名

- 有文件 → 从文件名提取（去掉扩展名）
- 无文件 → 必须向用户询问

#### 3b. 服务选择

根据文件解析结果智能建议：
- 检测到 Gerber → 建议开启 PCB 制板
- 检测到 BOM + 坐标 → 建议开启 SMT 贴片
- 用户明确说只要 PCB / 只要 SMT / 全要 → 按用户意图

#### 3c. PCB 工艺参数（pcb_enabled=true 时）

| 参数 | MCP 字段 | 默认值 | 允许值 |
|------|----------|--------|--------|
| PCB 数量 | `pcb_qty` | 5 | ≥5，5 的倍数 |
| 板层数 | `board_layers` | 2 | 1/2/4/6/8/10/12/14/16 |
| 板材 | `board_material` | FR4 | FR4/铝基板/Rogers/CEM-1 |
| 板厚(mm) | `board_thickness` | 1.6 | 0.4/0.6/0.8/1.0/1.2/1.6/2.0 |
| 外层铜厚 | `copper_thickness` | 1oz | 1oz/2oz/2.5oz/3.5oz |
| 内层铜厚 | `inner_copper_thickness` | 0.5oz | 0.5oz/1oz/2oz（仅 ≥4 层时有效） |
| 表面处理 | `surface_finish` | 有铅喷锡 | 有铅喷锡/无铅喷锡/沉金/OSP/镀金 |
| 阻焊颜色 | `solder_mask_color` | 绿色 | 绿色/红色/黄色/蓝色/白色/哑黑色/嘉立创紫 |
| 字符颜色 | `silkscreen_color` | 自动 | 白色/黑色（阻焊=白色时字符=黑色，否则字符=白色） |
| 最小线宽(mm) | `min_line_width` | 0.3 | 0.1/0.15/0.2/0.25/0.3 |
| 最小孔径(mm) | `min_hole_size` | 0.3 | 0.2/0.25/0.3/0.35/0.4 |
| 过孔处理 | `via_process` | 盖油 | 盖油/开窗/树脂塞孔 |
| 阻抗控制 | `impedance_control` | false | true/false |
| 阻抗值 | `impedance_value` | - | 文本，如 "50Ω"（仅阻抗控制开启时） |
| 特殊工艺 | `special_process` | - | 文本 |

> 如果上传文件返回了 `pcbParameters.layers`，自动设为 `board_layers` 默认值。

#### 3d. SMT 工艺参数（smt_enabled=true 时）

| 参数 | MCP 字段 | 默认值 | 允许值 |
|------|----------|--------|--------|
| SMT 数量 | `smt_qty` | 5 | ≥1，≤ pcb_qty（PCB 开启时） |
| 贴装面 | `smt_mounting_side` | TOP_ONLY | TOP_ONLY/BOTTOM_ONLY/BOTH_SIDES |
| 元器件来源 | `smt_component_source` | 邮寄 | 邮寄/代购/部分代购 |
| 客供补差 | `smt_customer_supply_fallback` | false | true=客供优先免费，不足按平台料价补差；false=纯客供全免费 |
| 特殊要求 | `smt_special_requirement` | - | 文本 |

> 如果上传文件返回了 `detectedMountingSide`，自动设为 `smt_mounting_side` 默认值。

#### 3e. DIP 插件参数（dip_enabled=true 时）

| 参数 | MCP 字段 | 默认值 | 允许值 |
|------|----------|--------|--------|
| DIP 数量 | `dip_quantity` | 1 | ≥1，≤ smt_qty（SMT 开启时） |
| 插件方式 | `dip_method` | 手工插件 | 手工插件/波峰焊 |
| 元器件来源 | `dip_component_source` | 邮寄 | 邮寄/代购 |
| 特殊要求 | `dip_special_requirement` | - | 文本 |

#### 3f. 其他服务

| 参数 | MCP 字段 | 默认值 | 说明 |
|------|----------|--------|------|
| 钢网服务 | `stencil_service` | false | 独立钢网服务（不含贴片） |
| 三防漆 | `conformal_coating` | false | 三防漆涂覆 |
| 组装测试 | `assembly_test` | false | PCBA 功能测试 |
| 测试方案 | `test_plan` | - | 文本（仅组装测试开启时） |

**钢网参数自动计算**（SMT 开启时，传给 `stencil_required` + `stencil_frame_size`）：

| 场景 | `stencil_required` | `stencil_frame_size` |
|------|---------------------|----------------------|
| SMT 关闭 | false | - |
| SMT 开启 + 样品（数量 < 20） | true | 钢网片 |
| SMT 开启 + 批量（数量 ≥ 20） | true | ""（空，由计价引擎决定） |

#### 3g. 交期与备注

| 参数 | MCP 字段 | 默认值 | 允许值 |
|------|----------|--------|--------|
| 交期 | `delivery_time` | 96 | 24（24小时加急）/48/96（常规） |
| 备注 | `remark` | - | 文本，≤500 字 |

> 用户说"加急" → delivery_time="24"；"明天要" → delivery_time="24"；"常规" → 96

#### 3h. 优惠券（可选）

下单前可查询可用优惠券：

1. `list_my_coupons(status="UNUSED")` → 展示用户的未使用优惠券
2. 用户选择后，将 `coupon_id` 传给 `create_order`

> 优惠券是否可用取决于订单金额是否满足门槛，后端会在下单时校验。

### Step 4：参数校验

调用 `create_order` 前，检查以下约束：

| 校验项 | 规则 | 不通过时处理 |
|--------|------|-------------|
| 产品名 | 非空 | 追问用户 |
| 服务启用 | 至少一项为 true | 引导用户选择 |
| PCB 数量 | ≥5 且为 5 的倍数 | 调整为最近合法值 |
| SMT 数量 | ≥1 | 追问 |
| DIP 数量 | ≥1 | 追问 |
| SMT ≤ PCB | PCB+SMT 都开启时 | 提示用户调整 |
| DIP ≤ SMT | SMT+DIP 都开启时 | 提示用户调整 |
| 贴装面 | SMT 开启时必填 | 追问 |
| 手机号 | 手动地址时 11 位 | 追问正确号码 |

### Step 5：创建订单

**操作**: `create_order`

**必传参数**:

| 参数 | 说明 |
|------|------|
| `product_name` | 产品名 |

**文件参数**（有文件时，从 Step 2 返回值填入）:

| 参数 | 来源 |
|------|------|
| `file_id` / `file_key` | upload 返回 |
| `gerber_file_id` / `gerber_file_path` | upload 返回 |
| `bom_file_id` / `bom_file_path` | upload 返回 |
| `coordinate_file_id` / `coordinate_file_path` | upload 返回 |
| `pcb_file_id` / `pcb_file_path` | upload 返回 |

**地址参数**（二选一）:

- 用已保存地址: `shipping_address_id`
- 手动填写: `receiver_name` + `receiver_phone` + `shipping_province` + `shipping_city` + `shipping_district` + `shipping_address`

**优惠参数**（可选）:

- `coupon_id`: 用户选择的优惠券 ID

> `order_type`（Y/P）无需传递，后端按数量自动判断：<20 样品，≥20 批量。

### Step 6：自检

**操作**: `get_order_detail(order_id)`

验证：
1. `productName` 正确
2. 文件关联已填充（有文件时）：`gerberFile`/`bomFile`/`coordinateFile` 非空
3. 服务开关正确：`pcbEnabled`/`smtEnabled`/`dipEnabled`
4. 地址正确
5. `deliveryTime` 正确

**展示给用户**:
- 订单号 / 订单 ID
- 产品名
- 启用的服务（PCB/SMT/DIP）
- 数量
- 交期
- 收货地址
- 状态（通常为 DRAFT 或 PENDING）

### Step 7：支付（可选）

订单创建后，如果用户想立即支付：

1. `create_payment(order_id, payment_method="WECHAT")` → 返回支付链接/二维码
2. 将支付链接展示给用户
3. 用户支付后可选：`query_payment_status(payment_id)` 确认到账

> 月结客户无需在线支付，跳过此步。

---

## Step 8：返单流程

对历史订单快速返单，无需重新上传文件和配置工艺。

**操作**: `create_repeat_order`

**必传**: `original_production_code`（原订单生产代号，从 `get_order_detail` 获取）

**可选**:

| 参数 | 说明 |
|------|------|
| `pcb_quantity` | 新 PCB 数量（不传则沿用） |
| `smt_quantity` | 新 SMT 数量（不传则沿用） |
| `dip_enabled` | 是否继续 DIP（false=停做，null/true=沿用） |
| `dip_quantity` | 新 DIP 数量 |
| `required_date` | 交期（ISO 格式，如 "2026-10-15T00:00:00"） |
| `use_same_bom` | 是否沿用原 BOM/文件（默认 true） |
| `remark` | 备注 |

**流程**:
1. 用户提供原订单号或生产代号
2. `get_order_detail(order_id)` 获取 `productionCode`
3. 询问是否调整数量/交期
4. `create_repeat_order(...)`
5. 自检（同 Step 6）

---

## Step 9：草稿恢复流程

1. `list_order_drafts()` → 展示草稿列表
2. 用户选择一份草稿
3. 从草稿数据中提取参数，走 Steps 1-5 正常创建
4. 创建成功后 `delete_order_draft(draft_id)` 清理

---

## 开票流程（可选，订单完成后）

### 开票资料管理

| 操作 | MCP 工具 |
|------|----------|
| 查开票资料列表 | `list_billing_infos` |
| 查默认开票资料 | `get_default_billing_info` |
| 新建开票资料 | `create_billing_info(company_name, tax_number, invoice_type, ...)` |
| 修改开票资料 | `update_billing_info(billing_info_id, billing_data)` |
| 设默认 | `set_default_billing_info(billing_info_id)` |
| 删除 | `delete_billing_info(billing_info_id)` |

**发票类型**: `NORMAL`=增值税普通发票 / `SPECIAL`=增值税专用发票（专票需银行信息）

### 开票申请

| 操作 | MCP 工具 |
|------|----------|
| 查可开票订单 | `get_invoiceable_orders` |
| 创建开票申请 | `create_invoice_application(invoice_type, billing_info_id, order_ids)` |
| 查开票记录 | `list_invoice_applications` |
| 取消开票 | `cancel_invoice_application(application_id)` |

---

## 错误处理

| 错误 | 处理 |
|------|------|
| 401 未认证 | 引导用户完成 OAuth 登录 |
| 文件过大（>100MB） | 提示压缩或拆分 |
| 上传文件未识别 Gerber | 检查 zip 内文件命名，确保 .gbr/.gbl 等标准后缀 |
| 产品名为空 | 追问用户 |
| SMT > PCB | 提示调整数量关系 |
| DIP > SMT | 提示调整数量关系 |
| 手机号格式错 | 追问 11 位正确号码 |
| 优惠券锁定失败 | 展示错误原因，可能券已过期或不满足门槛 |
| 内网 URL 不可访问 | 改用 upload_order_file(file_base64=...) 或分块上传 |

---

## 关联工具速查

| 场景 | MCP 工具 |
|------|----------|
| 查我的地址 | `list_addresses` |
| 新建地址 | `create_address` |
| 上传文件（统一入口） | `upload_order_file` |
| 大文件分块上传（>10MB 备选） | `start_chunked_upload` → `upload_file_chunk` → `complete_chunked_upload` |
| 创建订单 | `create_order` |
| 查订单详情 | `get_order_detail` |
| 查订单进度 | `get_order_timeline` |
| 查订单列表 | `list_orders` |
| 确认订单 | `confirm_order` |
| 取消订单 | `cancel_order` |
| 返单 | `create_repeat_order` |
| 修改收货地址 | `update_order_shipping_address` |
| 保存草稿 | `save_order_draft` |
| 查草稿列表 | `list_order_drafts` |
| 删草稿 | `delete_order_draft` |
| 查可用优惠券（领券中心） | `list_available_coupons` |
| 领券 | `claim_coupon` |
| 我的优惠券 | `list_my_coupons` |
| 下单可用券 | `list_usable_coupons` |
| 查开票资料 | `list_billing_infos` |
| 新建开票资料 | `create_billing_info` |
| 查可开票订单 | `get_invoiceable_orders` |
| 申请开票 | `create_invoice_application` |
| 创建支付 | `create_payment` |
| 查支付状态 | `query_payment_status` |
| 查订单支付记录 | `get_payments_by_order` |
| 查原始 BOM | `get_original_bom` |
| 查匹配 BOM | `get_matched_bom` |
| 查我的资料 | `get_my_profile` |
| 查订单统计 | `get_order_statistics` |
