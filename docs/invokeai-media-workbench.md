# StoryIP Forge × InvokeAI｜最小媒体工作台合同（V0.18 Draft）

> 目标：不自研图库、审批 UI、外部生图 Provider、持久工作流引擎。StoryIP Forge 只负责创作判断、Prompt/Reference 编译和最终资产注册；InvokeAI 负责候选媒体的生成/导入、图库、Board、Metadata 与 Star 审批。

## 1. 设计约束

按优先级同时满足：

1. **最简架构**：V0.18 只引入一个外部工作台 InvokeAI；不引入 ComfyUI、SwarmUI、LiteLLM、n8n/Activepieces。
2. **成本优先**：默认先尝试宿主环境可用的内置生图能力；只有技术调用失败时才让用户选择手工会员生成或付费 API。
3. **操作便捷**：所有候选图最终进入 InvokeAI；用户只需在 Gallery/Board 看图并 Star，`Star = APPROVED`。

## 2. 职责边界

### StoryIP Forge

负责：

- 读取正式上游资产与角色/场景约束；
- 形成唯一 `generation_spec`（任务、Prompt、Reference、锁定项、允许变化项）；
- 优先尝试宿主内置图片生成能力；
- 内置生成发生**技术错误**时询问用户：`manual_web` / `invoke_external_api` / `cancel`；
- 将外部候选图导入 InvokeAI 时附加 StoryIP Metadata；
- 读取 Star 结果；
- 将已批准候选原子发布到 StoryIP Asset Registry。

不负责：

- 自建 Gallery、Board、图片预览、Compare、Star UI；
- 自建外部 Provider SDK；
- 自建图片数据库；
- 自建持久任务队列或通用 DAG；
- 自动操作无公开 API 的会员网页。

### InvokeAI

直接复用其现成能力：

- External Models：OpenAI / Gemini / BytePlus Seedream / Alibaba Cloud DashScope；
- Reference Images；
- 外部 API 生成结果自动写入 Gallery；
- `POST /api/v1/images/upload` 导入宿主内置生成或手工生成的图片；
- Board 创建与分组；
- 图片 Metadata；
- Star / Unstar；
- Gallery / Board 浏览与人工筛选；
- API 错误、限流与 Provider 参数适配。

## 3. 唯一用户工作流

```text
用户提出资产任务
      ↓
StoryIP 编译 generation_spec
      ↓
尝试 host_builtin_image
      ├─ 成功 ───────────────┐
      │                     │
      └─ provider_error     │
             ↓              │
        询问用户             │
      ┌──────┼──────┐       │
      │      │      │       │
 manual   API   cancel      │
      │      │              │
 网页生成  Invoke External   │
      │      │              │
      └──────┴──────────────┘
             ↓
        InvokeAI Board
             ↓
       看图 / 对比 / Star
             ↓
         Star = APPROVED
             ↓
      StoryIP Asset Registry
```

## 4. 生成失败的判定

只在**技术错误**时触发 Provider 选择：

- 工具/Provider 不可用；
- 认证或额度错误；
- 超时；
- 服务端错误；
- 宿主环境不提供内置生图能力。

以下情况不算 Provider 技术失败：

- 脸不像；
- 动作不准；
- 构图不理想；
- 手部错误；
- 服装细节漂移。

这些属于 `quality_unsatisfied`，继续在当前 Provider 上依据用户反馈生成下一轮；用户也可主动要求换 Provider。

## 5. Provider 策略

V0.18 不实现自动多 Provider fallback。

逻辑固定为：

```yaml
image_generation:
  primary: host_builtin_image
  on_primary_provider_error: ask_user
  choices:
    - manual_web
    - invoke_external_api
    - cancel
```

说明：

- `host_builtin_image` 是宿主能力标识，不绑定具体模型；宿主不存在该能力时视为 `provider_error`。
- `manual_web` 适用于即梦等会员网页。StoryIP 只输出 Prompt/Reference 包，绝不做网页自动点击。
- `invoke_external_api` 交给 InvokeAI 已有 External Model Provider；V0.18 不在 StoryIP 内重复实现 OpenAI、Seedream、Gemini、DashScope SDK。

## 6. InvokeAI Board 约定

最简原则：**一个 generation job 一个 Board**。

推荐命名：

```text
storyip__<project>__<asset-id>__r<round>
```

例如：

```text
storyip__demo__character.miaoqiansui.identity__r03
```

Board 只是候选工作区，不是正式 Asset Registry。

## 7. Candidate Metadata 最小字段

所有来源在进入 InvokeAI 时统一写入：

```json
{
  "storyip_job_id": "character.miaoqiansui.identity.r03",
  "storyip_asset_id": "character.miaoqiansui.identity",
  "storyip_asset_type": "character_identity",
  "storyip_round": 3,
  "storyip_source": "host_builtin_image",
  "storyip_prompt_rev": 3
}
```

`storyip_source` 可取：

- `host_builtin_image`
- `manual_web`
- `invoke_external_api`

Reference、完整 Prompt、Provider 参数继续由 StoryIP 工作目录持有；不把 Invoke Metadata 扩成第二个语义 Owner。

## 8. 审批规则

V0.18 只保留一个人工动作：

```text
Star = APPROVED
Unstar / 未 Star = Candidate
```

不增加 Reject、Redo、Master、Reference 等第二套审批按钮。

资产类型（Identity / Turnaround / Action / Expression / Scene / Prop）在任务创建时已经确定，审批时不再让用户重复选择。

用户对候选不满意时，直接回到 Codex/StoryIP 描述修改意见，生成新 Round。

## 9. StoryIP 只需三个薄适配动作

### A. `push_candidate_to_invoke`

用途：把 `host_builtin_image` 或 `manual_web` 产生的本地图片上传 InvokeAI。

直接使用 InvokeAI 原生接口：

```text
POST /api/v1/images/upload
```

传入：文件、`board_id`、`session_id`（可选）、StoryIP metadata。

### B. `generate_via_invoke`

用途：用户选择付费 API 时，使用 InvokeAI 已配置的 External Model 生成。

StoryIP 不直接实现具体厂商 SDK，只触发 InvokeAI 的既有 External Image Generation 工作流/队列。

### C. `sync_starred_assets`

用途：读取指定 StoryIP Board 中已 Star 的候选，校验 Metadata 后原子发布到 Asset Registry。

必须满足：

- 只接受当前 `storyip_job_id` 的候选；
- 同一需要唯一正式输出的 Job 若 Star 多张，则停止并要求人工只保留最终一张 Star；
- 发布前计算内容 hash；
- Registry 写入成功后才把 StoryIP job 标记为完成。

## 10. 直接复用的 InvokeAI API

已从 InvokeAI 主仓库确认的接口：

```text
POST /api/v1/boards/?board_name=<name>   # 创建 Board
GET  /api/v1/boards/                     # 列出 Board
POST /api/v1/images/upload               # 上传外部图片 + metadata + board_id
POST /api/v1/images/star                 # Star 图片
POST /api/v1/images/unstar               # Unstar 图片
GET  /api/v1/images/...                  # Gallery/图片查询
```

实际接入时以安装版本 OpenAPI schema 为准，不复制 InvokeAI 内部数据库实现。

## 11. V0.18 明确不做

- 不做 Forge Studio；
- 不做 SwarmUI；
- 不做 ComfyUI；
- 不做 LiteLLM；
- 不做 n8n / Activepieces；
- 不做浏览器自动化生成；
- 不做自动成本路由；
- 不做后台自动多 Provider fallback；
- 不做视频生成；
- 不做通用任务队列；
- 不复制 InvokeAI Gallery/Board 数据库。

只有出现对应真实故障后才增加组件。

## 12. 最小验收闭环

V0.18 第一项验收只测试：

```text
1. StoryIP 创建测试 generation job
2. 创建/找到对应 InvokeAI Board
3. 将一张外部 PNG 上传到该 Board，并带 StoryIP Metadata
4. 用户在 InvokeAI Gallery 中 Star 该图
5. StoryIP 查询到 Star 状态
6. StoryIP 将该文件复制/发布为测试 Asset Registry 条目
```

这 6 步通过前，不接任何正式付费 Provider。

第二项验收才测试：

```text
StoryIP → InvokeAI External Model → Seedream/OpenAI 等 → Gallery → Star → Registry
```

Provider 选择单独决策，不阻塞第一项验收。

## 13. 上游项目依据

- InvokeAI：Apache-2.0；提供 External Models、Gallery/Board、Upload、Metadata、Star API。
- InvokeAI External Models 当前原生覆盖 OpenAI、Gemini、BytePlus Seedream、Alibaba Cloud DashScope。
- 新 Provider 仅在 InvokeAI 未覆盖且确有生产需求时，优先按 InvokeAI 官方 External Provider adapter 机制扩展，不在 StoryIP 重写一套。
