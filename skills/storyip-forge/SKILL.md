---
name: storyip-forge
description: 初始化、继续、恢复和路由“小说×IP漫剧”项目；维护固定目录、最小状态、单一Owner、人工Gate与最小上下文。明确专业任务必须路由到对应 Skill，主控不代替专业 Skill 做创作判断。
---

# 小说×IP漫剧主控

本 Skill 只负责工程路由，不拥有故事、表演、资产、声音、镜头、生成或剪辑专业判断。

## 每次请求

1. 使用用户明确的项目路径；否则从当前目录向上寻找最近 `project.yaml`。
2. 验证项目状态，再读取当前 cursor。
3. 用户明确请求优先；否则按 `state.cursor` 路由到最小专业 Skill。
4. 专业 Skill 只读其输入合同；正式写入前先冻结输入，再原子发布。
5. 一个阶段实际完成并进入下一阶段时，由主控显式推进 cursor。Runner 不自行发明工作流推进。

## 路由

| 意图 | Skill |
|---|---|
| 全文解剖、故事架构、情节采选、迁移重构、改编总纲 | `$story-adaptation` |
| 单集契约 | `$episode-design` |
| 正式剧本 | `$screenplay-writing` |
| 表演、调度、动作节奏 | `$directing` |
| 角色 / 场景 / 道具 / 稳定声音身份 | `$asset-design` |
| native / controlled、逐句声音表演 | `$sound-design` |
| `shots.jsonl` | `$shot-design` |
| Prompt 编译、预检、样片、批量、重试 | `$generation-orchestration` |
| KEEP / TRIM / REDO、Timeline、成片 | `$editing` |
| 四个高价值独立审查点 | `$quality-review` |

没有独立的“预告样片 Skill”或“批量生成 Skill”；二者是生成 Runner 模式。最终成片检查属于 `$quality-review` 的 final checkpoint。

## 持久 Gate

- **Story**：Reviewer 后仍需人工确认。
- **Episode Package**：`episode.md + screenplay.md` 经独立 Reviewer 通过后统一接受。
- **生产路线**：正式样片验证后人工确认；仅系统性路线变化重置。

Reviewer 不能越权接受或推进上述 Gate。

## Owner 与写入

正式语义权威只有：`story.md`、Episode Package、`shots.jsonl`、`timeline.yaml`，以及跨项目 IP Asset Registry。`.work/` 是可删除工作缓存；`.runtime/` 是程序运行数据；`deliverables/` 是派生输出。

每个正式写入遵循：

```text
最小读取集 → snapshot → 专业判断 → 结构检查 → Owner检查 → 原子发布
```

## 固定原则

- 一个语义事实一个 Owner；下游不得静默改写上游。
- Reviewer = READ + ANALYZE + REPORT。
- 程序只保证结构和执行事实，不代替创作判断。
- Prompt / Preflight / Provider 参数 / 候选不是语义 Artifact。
- 质量优先；成本只在预期质量接近的路线之间决定。
- 新机制只有对应真实故障并证明有价值时才进入主干。
- 现有结构解决不了具体 bug 之前，不新增通用 DAG、队列或工作流引擎。
