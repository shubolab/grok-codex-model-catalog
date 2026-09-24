# grok-codex-catalog

给 Codex 用的 **最小** Grok 4.x 模型目录（`model_catalog_json`）。一份文件里放 `grok-4.7`、`grok-4.6`、`grok-4.5`，选择器里可以切换。

本仓库不是代理、不是鉴权配置、也不是把 GPT-5.2 那两万多字系统提示克隆过来。它只提供 Codex CLI / Desktop 在自定义 Responses 中转上识别这些 slug 所必需的字段。

适用场景：你已经有 OpenAI-compatible 的 Responses 网关（例如自建中转），`config.toml` 里 `model` 要在这三条之间切换，但 Codex 内置目录里没有这些 slug。

## 为什么需要它

Codex 0.153.x 的 `model_catalog_json` 是 **严格 serde 必填字段**，不是“随便写个 slug 就能过”。

- 只写 `slug` / `display_name` 会直接报缺字段。
- `base_instructions` 和 `model_messages.instructions_template` **二选一即可**。不必两份都塞。
- 短身份说明就够。不必复制内置 GPT 模型那份超长 instructions。
- 这份 JSON **会替换** 内置模型列表。三条都在同一个 `models` 数组里，`/model` 里就会同时出现。

本仓库用 `codex debug models -c model_catalog_json=...` 实测过。

## 仓库里有什么

| 文件 | 作用 |
| --- | --- |
| `grok-4.x-catalog.json` | `grok-4.7` / `grok-4.6` / `grok-4.5` |
| `README.md` | 说明、安装、推理档位 |
| `LICENSE` | MIT |

不包含 token、`config.toml`、中转地址或任何密钥。

## 安装（你自己放到合适的位置）

1. 把 `grok-4.x-catalog.json` 拷到稳定路径，例如：

```text
%USERPROFILE%\.codex\grok-4.x-catalog.json
```

2. 在 `%USERPROFILE%\.codex\config.toml` 里加（Windows 路径用双反斜杠）：

```toml
model = "grok-4.7"
model_catalog_json = "C:\\Users\\<you>\\.codex\\grok-4.x-catalog.json"
```

`model` 也可以写成 `grok-4.6` 或 `grok-4.5`。三条都在同一份 catalog 里，之后用 `/model` 切换即可。

3. 重启 Codex Desktop / CLI。

4. 中转鉴权仍走你现有的 `model_provider` / bearer token。catalog **不是** 登录配置。

校验：

```powershell
codex debug models -c "model_catalog_json=C:/Users/<you>/.codex/grok-4.x-catalog.json"
```

成功时应看到 3 个模型，slug 为 `grok-4.7`、`grok-4.6`、`grok-4.5`。

## 字段说明（只保留必要项）

Codex 解析器真正必填的是这些：

- `slug`, `display_name`
- `supported_reasoning_levels`
- `shell_type`
- `visibility`
- `supported_in_api`
- `priority`
- `support_verbosity`
- `truncation_policy`
- `experimental_supported_tools`
- `base_instructions` **或** `model_messages.instructions_template`

其余是可选增强。本文件额外写了这些，因为它们会改变默认行为，而不是为了“看起来完整”：

- `default_reasoning_level = "high"`：对齐 xAI 官方默认。
- `context_window` / `max_context_window = 500000`：三条都对齐 xAI 公开的 500K。
- `default_verbosity = "high"`：方便 agent 工作。
- `include_skills_usage_instructions` / `include_plugin_usage_instructions = true`：保留 Codex skills / plugins 提示。

**刻意不写：**

- 超长 `base_instructions` / `model_messages` 克隆
- OpenAI 后端才有的 lite / search tool / Fast-Priority speed tiers / `ultra` / `max` effort
- 把上下文写成传闻中的 2M（官方公开值是 500K）

`base_instructions` 只有三行身份说明。Codex 运行时自己会补 tool / sandbox 规则；catalog 不必再抄一份。

## 推理档位

三条都不能关掉推理，默认都是 `high`。

| 模型 | Codex 里列出的档位 |
| --- | --- |
| `grok-4.7` | `low` / `medium` / `high` / `xhigh` |
| `grok-4.6` | `low` / `medium` / `high` / `xhigh` |
| `grok-4.5` | `low` / `medium` / `high` / `xhigh` |

`xhigh` 从 `grok-4.6` 起才是独立档位。xAI 对 `grok-4.5` 收到的 `xhigh` 按 `high` 处理，请求不会因此 400，所以这里仍列出来，避免切到 4.5 时选择器把这一档拿掉。

| 档位 | 什么时候用 |
| --- | --- |
| `low` | 短循环、简单改文件、低延迟 |
| `medium` | 长上下文分析、一般编码 |
| `high` | 日常默认（官方默认） |
| `xhigh` | 很难的 agent 任务，质量优先于速度 |

在 `config.toml` 里用 `model_reasoning_effort` 覆盖，例如：

```toml
model_reasoning_effort = "xhigh"
```

不要在这份 catalog 里加 `ultra` / `max`，除非你的中转明确会把它们映射成 xAI 认识的档位。

## 中转注意

- 网关必须走 **Responses**（`wire_api = "responses"`），不是只实现了 Chat Completions 就够。
- 不要打开 OpenAI-only 的 lite / web search / Fast 档位。
- catalog 替换内置列表后，模型选择器里就是这三条。
- 上下文先按 500K 配。只有当你确认网关真的转发更大窗口时再改。
- `grok-4.7` 在 Responses API 上总会返回 `reasoning.encrypted_content`。多轮对话要把这些 reasoning item 原样带回下一次请求的 `input`，中转不要剥掉。

## 参考

- [xAI Grok 4.7](https://docs.x.ai/developers/grok-4-7)
- [xAI Grok 4.6](https://docs.x.ai/developers/grok-4-6)
- [xAI Grok 4.5](https://docs.x.ai/developers/models/grok-4-5)
- [xAI Reasoning](https://docs.x.ai/developers/model-capabilities/text/reasoning)
- [Codex config reference](https://developers.openai.com/codex/config-file/config-reference.md)

## License

MIT
