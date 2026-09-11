# grok-codex-catalog

给 Codex 用的 **最小** Grok 4.6 模型目录（`model_catalog_json`）。

本仓库不是代理、不是鉴权配置、也不是把 GPT-5.2 那两万多字系统提示克隆过来。它只提供 Codex CLI / Desktop 在自定义 Responses 中转上识别 `grok-4.6` 所必需的字段。

适用场景：你已经有 OpenAI-compatible 的 Responses 网关（例如自建中转），`config.toml` 里 `model = "grok-4.6"`，但 Codex 内置目录里没有这个 slug，需要一份能被当前 Codex 解析器接受的 catalog。

## 为什么需要它

Codex 0.153.x 的 `model_catalog_json` 是 **严格 serde 必填字段**，不是“随便写个 slug 就能过”。

- 只写 `slug` / `display_name` 会直接报缺字段。
- `base_instructions` 和 `model_messages.instructions_template` **二选一即可**。不必两份都塞。
- 短身份说明就够。不必复制内置 GPT 模型那份超长 instructions。
- 这份 JSON **会替换** 内置模型列表，所以仓库只放 `grok-4.6` 一条。

本仓库用 `codex debug models -c model_catalog_json=...` 实测过：约 1.4 KB 即可通过解析。

## 仓库里有什么

| 文件 | 作用 |
| --- | --- |
| `grok-4.6-catalog.json` | 最小可解析 catalog |
| `README.md` | 说明、安装、推理档位 |
| `LICENSE` | MIT |

不包含 token、`config.toml`、中转地址或任何密钥。

## 安装（你自己放到合适的位置）

1. 把 `grok-4.6-catalog.json` 拷到稳定路径，例如：

```text
%USERPROFILE%\.codex\grok-4.6-catalog.json
```

2. 在 `%USERPROFILE%\.codex\config.toml` 里加（Windows 路径用双反斜杠）：

```toml
model = "grok-4.6"
model_catalog_json = "C:\\Users\\<you>\\.codex\\grok-4.6-catalog.json"
```

3. 重启 Codex Desktop / CLI。

4. 中转鉴权仍走你现有的 `model_provider` / bearer token。catalog **不是** 登录配置。

校验：

```powershell
codex debug models -c "model_catalog_json=C:/Users/<you>/.codex/grok-4.6-catalog.json"
```

成功时应看到 `n_models=1`、slug `grok-4.6`。

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
- `context_window` / `max_context_window = 500000`：对齐 xAI 公开的 500K。
- `default_verbosity = "high"`：方便 agent 工作。
- `include_skills_usage_instructions` / `include_plugin_usage_instructions = true`：保留 Codex skills / plugins 提示。

**刻意不写：**

- 超长 `base_instructions` / `model_messages` 克隆
- OpenAI 后端才有的 lite / search tool / Fast-Priority speed tiers / `ultra` / `max` effort
- 把上下文写成传闻中的 2M（官方公开值是 500K）

`base_instructions` 只有三行身份说明。Codex 运行时自己会补 tool / sandbox 规则；catalog 不必再抄一份。

## 推理档位

xAI 官方：`grok-4.6` 支持 `low` / `medium` / `high` / `xhigh`，**默认 `high`**，推理不能关掉。

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
- catalog 替换内置列表后，模型选择器里通常只剩 `Grok 4.6`。
- 上下文先按 500K 配。只有当你确认网关真的转发更大窗口时再改。

## 参考

- [xAI Grok 4.6](https://docs.x.ai/developers/grok-4-6)
- [xAI Reasoning](https://docs.x.ai/developers/model-capabilities/text/reasoning)
- [Codex config reference](https://developers.openai.com/codex/config-file/config-reference.md)

## License

MIT
