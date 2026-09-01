# skills

个人 WorkBuddy Skills 收藏（项目级）。每个子目录是一个独立 skill，目录名即 skill 名，内含 `SKILL.md`。

## 目录

- [`customer-risk-report-gen/`](./customer-risk-report-gen/) — 对乐享知识库「客户评估清单」中「未生成」客户，逐家生成单客户维度的信用与法律风险评估报告(HTML)。支持：①仅本地生成（只需企查查）；②上传乐享知识库并同步清单（需额外绑定乐享数据库）。调用时先校验连接器在线状态——企查查为必要条件，乐享数据库按用户选择决定是否必须。

## 使用

- **同工作区团队**：把对应 skill 目录复制到团队项目的 `.workbuddy/skills/`，协作者打开即自动加载。
- **个人**：复制到 `~/.workbuddy/skills/`。
- **分享**：本仓库设为 private，按需添加 GitHub 协作者（Settings → Collaborators）即可共享给指定的人；skill 文件不含任何凭据，连接器需各自手动授权。

> ⚠️ 部分 skill 内含指向特定知识库 / 工作区的标识（entry_id、company_from、字段 ID 等），跨环境使用前请按需替换。
