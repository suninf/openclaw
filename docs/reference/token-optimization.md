---
summary: "降低 OpenClaw token 用量和 API 成本的配置指南"
read_when:
  - 为成本敏感的部署优化 token 消耗
  - 配置二次开发的省 token 预设
  - 设置模型分层路由和上下文裁剪
title: "Token 优化指南"
---

# Token 优化指南

本指南涵盖六项配置策略，用于降低 OpenClaw token 用量和 API 成本。所有设置均通过网关配置应用，无需修改源码。

关于 OpenClaw 如何构建 prompt 上下文和报告 token 用量的背景知识，参见 [Token 用量与成本](/reference/token-use)。

## 快速开始

将省 token 配置模板应用到网关配置：

```bash
openclaw config patch examples/configs/token-frugal-moderate.yaml
```

提供两套模板：

- **`examples/configs/token-frugal-moderate.yaml`** — 推荐档，平衡成本节省与响应质量
- **`examples/configs/token-frugal-aggressive.yaml`** — 激进档，最大程度节省，部分质量取舍

每套模板涵盖以下全部六个优化方向，请根据实际场景调整数值。

---

## 1. 系统 Prompt 瘦身

系统 prompt 包含工作区 bootstrap 文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md`、`MEMORY.md`）。它们在每轮对话中注入，是最大的固定开销。

### 配置项

| 配置项                                   | 默认值   | 推荐档              | 激进档              | 说明                                  |
| ---------------------------------------- | -------- | ------------------- | ------------------- | ------------------------------------- |
| `agents.defaults.bootstrapMaxChars`      | 12000    | 5000                | 3000                | 单个 bootstrap 文件截断前的最大字符数 |
| `agents.defaults.bootstrapTotalMaxChars` | 60000    | 30000               | 15000               | 所有 bootstrap 文件的总字符上限       |
| `agents.defaults.contextInjection`       | "always" | "continuation-skip" | "continuation-skip" | 何时重新注入 bootstrap 上下文         |

### 权衡

- `bootstrapMaxChars` 低于 3000 可能截断 `AGENTS.md` 中的关键规则。
- `contextInjection: "continuation-skip"` 在安全续写轮次跳过重新注入，节省后续对话的 token。
- 子 Agent 和 Cron 任务已有内置精简白名单（仅保留 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`），无需额外配置。
- 要减少 `SOUL.md` / `USER.md` 开销，需精简文件内容本身；配置仅控制截断上限。

### 示例

```yaml
agents:
  defaults:
    bootstrapMaxChars: 5000
    bootstrapTotalMaxChars: 30000
    contextInjection: "continuation-skip"
```

---

## 2. 模型分层路由

将不同任务类型路由到不同成本档位的模型。复杂任务用高能力模型，常规任务用低成本模型。

### 配置项

| 配置项                             | 默认值       | 推荐档               | 激进档                     | 说明                      |
| ---------------------------------- | ------------ | -------------------- | -------------------------- | ------------------------- |
| `agents.defaults.subagents.model`  | (继承主模型) | `<your-cheap-model>` | `<your-cheap-model>`       | 子 Agent 使用的模型       |
| `agents.defaults.heartbeat.model`  | (继承主模型) | `<your-cheap-model>` | `<your-lowest-cost-model>` | 心跳检查使用的模型        |
| `agents.defaults.compaction.model` | (继承主模型) | `<your-cheap-model>` | `<your-cheap-model>`       | Compaction 摘要使用的模型 |

### 推荐三层架构

| 层级 | 适用场景             | 示例模型                                                                  |
| ---- | -------------------- | ------------------------------------------------------------------------- |
| 复杂 | 主 Agent、创意任务   | `anthropic/claude-sonnet-4-6`、`openai/gpt-5.4`                           |
| 常规 | 子 Agent、compaction | `openrouter/deepseek/deepseek-chat-v3-0324`、`anthropic/claude-haiku-4-5` |
| 后台 | 心跳、空闲检查       | `ollama/qwen3:4b`、本地模型                                               |

### 权衡

- 低成本模型可能生成质量较低的摘要（compaction）或较简单的子 Agent 结果。
- 本地 Ollama 模型需运行 Ollama 服务并配置 provider。
- `openrouter:free` 别名自动路由到配置中首个 `:free` 模型；`openrouter:auto` 启用自动路由。

### 示例

```yaml
agents:
  defaults:
    model:
      primary: "<your-main-model>"
    subagents:
      model: "<your-cheap-model>"
    heartbeat:
      model: "<your-lowest-cost-model>"
    compaction:
      model: "<your-cheap-model>"
```

子 Agent 模型解析优先级：显式覆盖 → `agent.subagents.model` → `agents.defaults.subagents.model` → 主模型。

---

## 3. 上下文管理与裁剪

控制对话历史的增长与裁剪方式。上下文裁剪自动清除过期工具输出；compaction 对旧对话轮次进行摘要。

### 配置项

| 配置项                                                 | 默认值 | 推荐档      | 激进档      | 说明                                |
| ------------------------------------------------------ | ------ | ----------- | ----------- | ----------------------------------- |
| `agents.defaults.contextPruning.mode`                  | "off"  | "cache-ttl" | "cache-ttl" | 启用基于缓存 TTL 的裁剪             |
| `agents.defaults.contextPruning.ttl`                   | —      | "1h"        | "30m"       | 缓存 TTL 过期后执行裁剪             |
| `agents.defaults.contextPruning.keepLastAssistants`    | —      | 3           | 2           | 始终保留最近 N 轮 assistant 消息    |
| `agents.defaults.contextPruning.softTrimRatio`         | —      | 0.7         | 0.5         | 上下文超过此比例时软裁剪            |
| `agents.defaults.contextPruning.hardClearRatio`        | —      | 0.85        | 0.75        | 上下文超过此比例时硬清除            |
| `agents.defaults.compaction.reserveTokensFloor`        | 0      | 20000       | 10000       | Compaction 后保留的最低 token 余量  |
| `agents.defaults.contextLimits.toolResultMaxChars`     | 16000  | 8000        | 4000        | 单个工具结果截断前的最大字符数      |
| `agents.defaults.contextLimits.postCompactionMaxChars` | 1800   | 1200        | 800         | Compaction 后上下文注入的最大字符数 |
| `memory.qmd.limits.maxInjectedChars`                   | —      | 6000        | 3000        | 每轮注入的 QMD 文本最大字符数       |
| `memory.qmd.limits.maxResults`                         | 6      | 4           | 3           | QMD 搜索最大结果数                  |
| `memory.qmd.limits.maxSnippetChars`                    | 700    | 500         | 300         | 单条 QMD 片段的最大字符数           |

### 权衡

- `contextPruning.mode: "cache-ttl"` 在 Anthropic provider 下效果最佳（设置 `ANTHROPIC_API_KEY` 时自动检测）。
- `toolResultMaxChars` 低于 4000 可能截断重要的工具输出（网页、文件内容）。
- `reserveTokensFloor` 确保 compaction 后模型有足够空间回复；10000 是安全下限。
- 工具结果的 30% 上下文窗口限制（`MAX_TOOL_RESULT_CONTEXT_SHARE`）是硬编码的，不可配置。

### 示例

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
      keepLastAssistants: 3
      softTrimRatio: 0.7
      hardClearRatio: 0.85
    compaction:
      reserveTokensFloor: 20000
    contextLimits:
      toolResultMaxChars: 8000
      postCompactionMaxChars: 1200

memory:
  qmd:
    limits:
      maxInjectedChars: 6000
      maxResults: 4
      maxSnippetChars: 500
```

---

## 4. Prompt 缓存

Prompt 缓存复用已处理的 token，降低输入成本。Anthropic 缓存读取成本约为正常输入 token 的 10%。

### 配置项

| 配置项                                           | 默认值              | 推荐档  | 激进档  | 说明                                |
| ------------------------------------------------ | ------------------- | ------- | ------- | ----------------------------------- |
| `agents.defaults.models.*.params.cacheRetention` | "short" (Anthropic) | "short" | "short" | 缓存 TTL："none"、"short" 或 "long" |

### 策略建议

以下为行为实践，非配置项：

- **保持 bootstrap 文件稳定。** 避免频繁编辑 `AGENTS.md`、`SOUL.md`、`TOOLS.md` — 修改会使缓存失效。
- **用心跳保持缓存热度。** 将 `heartbeat.every` 设为略低于缓存 TTL（如 55m 心跳 + 1h 缓存 TTL），可在空闲间隔保持缓存有效。
- **5 分钟内批量请求。** TTL 窗口内的请求缓存命中率最高。
- **默认使用 `"short"`。** `"long"`（1h）缓存写入成本更高，仅在长连续会话中有益。
- **非 Anthropic provider** 需显式设置 `cacheRetention`，不会自动应用。

### 示例

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-sonnet-4-6":
        params:
          cacheRetention: "short"
    heartbeat:
      every: "55m"
```

另见 [Prompt 缓存](/reference/prompt-caching)。

---

## 5. 工具和媒体限制

减少网页搜索、抓取、图片处理和媒体工具返回的数据量。

### 配置项

| 配置项                                            | 默认值 | 推荐档 | 激进档 | 说明                        |
| ------------------------------------------------- | ------ | ------ | ------ | --------------------------- |
| `tools.web.search.maxResults`                     | —      | 3      | 2      | 搜索最大结果数 (1-10)       |
| `tools.web.fetch.maxChars`                        | 20000  | 8000   | 4000   | 抓取内容最大字符数          |
| `tools.web.fetch.maxResponseBytes`                | 750000 | 300000 | 150000 | 截断前最大下载大小          |
| `agents.defaults.imageMaxDimensionPx`             | 1200   | 600    | 400    | 图片最大边长（像素）        |
| `agents.defaults.contextLimits.memoryGetMaxChars` | 12000  | 6000   | 3000   | memory_get 返回的最大字符数 |

### 权衡

- `imageMaxDimensionPx` 低于 600 可能降低 OCR 和截图识别准确率。
- `web.fetch.maxChars` 低于 4000 可能无法获取足够内容来提供有意义的回答。
- 媒体理解 `maxChars` 默认值（图片/视频 500）已经很低，通常无需进一步降低。

### 示例

```yaml
agents:
  defaults:
    imageMaxDimensionPx: 600
    contextLimits:
      memoryGetMaxChars: 6000

tools:
  web:
    search:
      maxResults: 3
    fetch:
      maxChars: 8000
      maxResponseBytes: 300000
```

---

## 6. 心跳与后台控制

心跳定期运行以保持会话活跃和缓存热度。降低心跳频率和范围可直接减少后台 token 消耗。

### 配置项

| 配置项                                           | 默认值  | 推荐档  | 激进档  | 说明                                   |
| ------------------------------------------------ | ------- | ------- | ------- | -------------------------------------- |
| `agents.defaults.heartbeat.every`                | "30m"   | "120m"  | "4h"    | 心跳运行间隔                           |
| `agents.defaults.heartbeat.target`               | "last"  | "none"  | "none"  | 心跳目标："last"、"none" 或频道 id     |
| `agents.defaults.heartbeat.activeHours.start`    | —       | "09:00" | "09:00" | 活跃时段开始（HH:MM，24 小时制）       |
| `agents.defaults.heartbeat.activeHours.end`      | —       | "22:00" | "18:00" | 活跃时段结束（HH:MM，24 小时制，不含） |
| `agents.defaults.heartbeat.activeHours.timezone` | "local" | "local" | "local" | 活跃时段时区                           |

### 权衡

- `target: "none"` 执行内部检查但不发送消息，节省输出 token。
- `activeHours` 将心跳限制在工作时段；窗口外的心跳被跳过。
- 将 `heartbeat.every` 设为低于缓存 TTL（如 55m vs 1h）可保持缓存热度。设置过高可能导致空闲后缓存失效。
- 过长间隔（4h+）可能降低通知驱动工作流的响应性。

### 示例

```yaml
agents:
  defaults:
    heartbeat:
      every: "120m"
      target: "none"
      activeHours:
        start: "09:00"
        end: "22:00"
        timezone: "local"
```

---

## 验证

应用省 token 配置后：

1. 使用 `/status` 和 `/usage tokens` 观察每轮 token 消耗变化。
2. 使用 `/context list` 或 `/context detail` 查看注入的 bootstrap 上下文大小。
3. 对比优化前后的 `input_tokens`、`cacheRead` 和 `output_tokens` 数据。
4. 运行 `openclaw doctor` 确认无配置兼容性问题。
5. 测试多轮对话，验证 `contextPruning` 和 `compaction` 工作正常。

## 相关文档

- [Token 用量与成本](/reference/token-use)
- [Prompt 缓存](/reference/prompt-caching)
- [网关配置](/gateway/configuration)
- [会话裁剪](/concepts/session-pruning)
