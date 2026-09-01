---
name: customer-risk-report-gen
displayName: 客户评估报告生成
description: 对乐享知识库「客户评估清单」中状态为"未生成"的客户，逐家生成单客户维度的信用与法律风险评估报告(HTML)。支持两种模式：①仅本地生成（只需企查查在线）；②上传乐享知识库并同步清单（需额外绑定乐享数据库）。调用时先校验两个连接器在线状态——企查查为必要条件，乐享数据库按用户选择决定是否必须。当用户提到"生成客户评估报告"、"处理客户评估清单"、"未生成客户"、"客户风险评估"、"更新客户清单"等意图时触发。
version: 1.0.0
author: 小虾米
agent_created: true
---

# 客户评估报告生成

对乐享知识库「客户评估报告 / 客户评估清单」中**状态 = 未生成**的客户，逐家生成**单客户维度**的信用与法律风险评估报告（HTML）。

## 触发条件

当用户表达以下意图时触发：
- "生成客户评估报告"、"处理客户评估清单"、"未生成客户"、"客户风险评估"
- "更新客户清单"、"把未生成的客户跑一遍"
- 泛指"按操作手册处理客户"

## 前置条件（调用时先校验，Agent 无法代做）

### 0. 连接器在线校验（第一步必做）
调用本 skill 后，先确认所需连接器是否已连接（WorkBuddy 左侧「连接器」面板状态为"已连接"）：
- **企查查（QCC）—— 必要条件**：必须在线。若离线，明确提示用户在「连接器」面板手动连接并授权「企查查」连接器，**连好之前不得继续任何数据拉取**。
- **乐享数据库（Lexiang）—— 条件必需**：是否必须取决于用户选择的输出模式（见第 1 步）。若用户选择"上传知识库"而它离线，引导用户在「连接器」面板绑定 / 授权「乐享数据库（Lexiang）」连接器后再继续。
- 任一**必需**连接器离线时，停止后续流程，给出简明连接指引；待用户确认已连接后再继续。

### 1. 确定输出模式（用户决策，二选一）
向用户确认报告产出方式：
- **A. 仅本地生成**：报告生成为本地 HTML 文件，不上传知识库。此模式**只需企查查在线**，乐享数据库可保持离线。
- **B. 上传知识库**：报告上传至乐享知识库并同步「客户评估清单」。此模式**必须**乐享数据库在线；若离线，回到第 0 步引导用户绑定。

### 2. 阅读 SOP（仅模式 B 需要）
若选择上传知识库，对话开局应先阅读知识库《客户评估报告生成操作手册(SOP)》（entry `73daf29a78234b0384169b32754fa055`，链接 `https://lexiangla.com/pages/73daf29a78234b0384169b32754fa055?company_from=c6dbc47643aa11f1ba7f5adeb259711b`），重点看第二部分「Agent 操作指引」。模式 A 可跳过——本 skill 已包含全部字段映射、payload 与踩坑要点。

## 防止输出偏移的铁律（务必遵守）

1. **单一客户维度**：每份报告只写当前客户自身信息，**绝不**跨客户对比、不放对比表。
2. **严格复用模板**：复制知识库内母版（嘉兴奥力弗报告 entry `0dfac20867424090b2f3851bafbb34fe`）的 CSS 与 DOM 结构——深蓝 `--brand:#1f4e79`、`.doc-head` + `.verdict`/`.score-badge` + `section h3 .num` 骨架。**禁止**混用其他封面/配色样式。
3. **评级标准**：A 优质 / B 中低风险 / C 中高风险 / D 高风险；**财务数据连续三年全部不公示 → 最高给 B 级**。
4. **链接必须是绝对 URL（仅上传知识库模式）**：清单「链接」字段的 `link` 与 `text` 都要写 `https://lexiangla.com/pages/{entry_id}?company_from=c6dbc47643aa11f1ba7f5adeb259711b`，相对路径点了无反应。

## 关键标识与字段映射

| 字段 | field_id | 类型 | 写入取值 / 说明 |
|---|---|---|---|
| 客户名称 | `4c775e8d` | text | 只读参考，不回写 |
| 状态 | `50bcf8e2` | single_select | `2c9fb5f2`=已生成 ／ `08b19aba`=未生成 |
| 生成日期 | `636cb2b5` | date_time | `{"date_time":{"date":"2026-09-01"}}` |
| 链接 | `ce308a1d` | url | link 与 text 均为绝对 URL（见下） |
| 评级 | `ee1a20ea` | single_select | `00261282`=D ／ `e70f9397`=B ／ `2d2212ca`=C |

固定参数：
- 知识库 `company_from = c6dbc47643aa11f1ba7f5adeb259711b`
- 清单 entry_id `96ea26e512a144c5bc897978988e1065`、smartsheet_id `2a9c11e27a704e78b905905e3eb3fe36`
- 知识库 root_entry `610ba950a4cc40e8ab52fa645dd7fe1a`

## 标准流程（逐家执行）

1. **校验连接器 + 确认模式**：见前置条件 0 与 1。
2. **拉取未生成项（仅模式 B）**：`smartsheet_list_records`（entry_id / smartsheet_id 见上），筛选 `50bcf8e2 = 08b19aba`。记录每家的 `record_id` 与客户名称。模式 A 则由用户直接提供客户名单。
3. **锁定主体**：`get_company_by_query`（searchKey = 客户名）。**多候选时必须让用户选定，勿自动取第一个。**
4. **拉企查查数据**（统一用信用代码作 searchKey，并行调用）：`get_company_registration_info` / `get_company_profile` / `get_shareholder_info` / `get_actual_controller` / `get_beneficial_owners` / `get_key_personnel` / `get_change_records` / `get_external_investments` / `get_branches` / `get_annual_reports` / `get_financial_data` / `get_listing_info`。
5. **补司法风险**：WebSearch（关键词组：司法案件 裁判文书 ／ 被执行人 失信 限高 ／ 股权冻结 行政处罚 ／ 关联方），用天眼查·启信宝·爱企查交叉验证。单一来源负面信息标注「待核实」。
6. **评级判定**：综合经营（社保人数年度变化最敏感）、法律（原告/被告比例比总数更重要）、财务，给 A/B/C/D，记录判读依据。
7. **生成报告 HTML**：复制母版文件改内容，保持结构与样式不变；章节顺序：评级摘要 → 基本信息 → 股权结构/实控人 → 经营状况 → 法律纠纷 → 关联方 → 风险清单分级 → 授信建议 → 数据来源/免责。
8. **产出（按模式分支）**：
   - **模式 A · 仅本地生成**：将报告 HTML 保存至本地工作区（如 `outputs/客户名_客户评估报告.html`），流程结束，不依赖乐享数据库。
   - **模式 B · 上传知识库**：`file_apply_upload`（PRE_SIGNED_URL）→ `curl -X PUT` 上传（签名中 `\u0026` 还原为 `&`，带 `Content-Type: text/html`）→ `file_commit_upload` 得到 entry_id；再以 `smartsheet_update_records` 按下方示例同步清单（状态/日期/链接/评级）。

## 清单同步 payload 示例（仅模式 B）

```json
{
  "entry_id": "96ea26e512a144c5bc897978988e1065",
  "smartsheet_id": "2a9c11e27a704e78b905905e3eb3fe36",
  "records": [
    {
      "record_id": "<清单中该客户的 record_id>",
      "fields": {
        "50bcf8e2": {"single_select": {"id": "2c9fb5f2"}},
        "636cb2b5": {"date_time": {"date": "2026-09-01"}},
        "ce308a1d": {"url": {
          "link": "https://lexiangla.com/pages/{entry_id}?company_from=c6dbc47643aa11f1ba7f5adeb259711b",
          "text": "https://lexiangla.com/pages/{entry_id}?company_from=c6dbc47643aa11f1ba7f5adeb259711b"
        }},
        "ee1a20ea": {"single_select": {"id": "e70f9397"}}
      }
    }
  ]
}
```

## 踩坑清单（务必避开）

- **① 清单链接打不开**：用了 `mcp.lexiang-app.com` 域名或缺 `company_from`，或 `link` 写成相对路径 `/pages/...`。正确：绝对 URL + `company_from`。
- **② 模板漂移**：曾因上传中途限流恢复被重新生成一套亮蓝渐变封面，与深蓝模板不一致。务必复制母版改内容。
- **③ 跨客户对比**：报告内只写单一客户，不放对比表。
- **④ 企查查无司法工具**：法律纠纷靠 WebSearch + 第三方平台交叉验证，单一来源须标「待核实」。
- **⑤ 乐享无删除接口**：重复/旧条目只能网页端手删；Agent 发现后提示用户处理，不要尝试用 MCP 删除。
- **⑥ 连接器离线**：调用前务必校验在线状态。企查查离线则整体不可进行；乐享离线仅在"上传知识库"模式阻塞，本地模式不受影响。

## 复核与收尾

- 模式 B 全部客户处理完后，**重列清单**确认：目标 record 状态均为「已生成」、链接均为绝对 URL 且可点击、评级正确。
- 若存在重复/旧条目，输出提示让用户去网页端手动删除（提供 entry_id）。
- 若流程变动（字段 ID、模板、company_from 等），同步更新 SOP 文档头部「最后更新」日期与本文档。

## 如何分享给其他团队成员

本 skill 已放在**项目级**目录（HYLINK 工作区的 `.workbuddy/skills/`）。只要协作者打开/拉取同一个工作区，就会**自动获得**此 skill，无需额外安装。

若需分享给**不在同一工作区**的人，任选其一：

1. **用户级（推荐给个人）**：把本文件夹（`customer-risk-report-gen/`）整体复制到对方的 `<用户主目录>/.workbuddy/skills/`。
2. **项目级（团队）**：复制到对方项目的 `<项目>/.workbuddy/skills/`。
3. **全公司（一般不必要）**：发布到内部市场 BuiltinMarket。

⚠️ **安全与授权提醒**：
- skill 文件**不含任何账号、密码、token**，它只是流程指令。
- 连接器（企查查 / 乐享数据库）仍需**每位使用者用自己的账号手动连接授权**——skill 无法也不应代存凭据。
- 知识库 SOP 已在乐享（团队共有），对方无需额外授权即可阅读；但向清单写入需该知识库的写入权限。
