# Codex + CPA + cc-switch：恢复内置生图与模型列表

记录一次 Windows 上的配置排查：CPA（CLIProxyAPI）已支持图片接口，但 Codex 聊天没有 `image_gen` 工具。修复后内置图片执行器成功出图，CPA 的模型也自动出现在 Codex 模型选择器中。

实测日期：2026-10-10。实测 Codex CLI/桌面后端版本：`0.162.0-alpha.2`，默认模型 `gpt-6-astra`。CPA 与 cc-switch 的具体版本未记录。这是针对该环境验证过的兼容配置，不保证所有版本或其他网关适用。

## 快速复现（Windows）

1. 先备份现有 Codex 和 cc-switch 配置。
2. 在 cc-switch 的 Codex 供应商里填入自己的 CPA 地址和 API Key，确认应用后 `$CODEX_HOME/auth.json`（默认 `%USERPROFILE%\.codex\auth.json`）含 `OPENAI_API_KEY`。
3. 把 [config.windows.toml](config.windows.toml) 放入供应商的 config 配置区，将 `https://YOUR-CPA-HOST/v1` 改为真实 CPA 地址。默认模型也可以换成 CPA 支持的模型。
4. 应用配置，完整退出并重新打开 Codex，再测试“生成一张风景图”；旧聊天未刷新时使用新聊天。

若直接编辑 `config.toml`，请合并已有 TOML 表，不要重复声明 `[features]` 等表。已有 `http_headers` 要合并。若已有 `features.code_mode = true` 这种布尔写法，应改为配置表后合并设置，不能同时保留两种写法。

配置不含密钥或固定用户名路径；认证命令自动使用 `CODEX_HOME`，未设置时使用当前用户的 `.codex`。这是 Windows 模板，macOS/Linux 需要替换认证命令。

## 必要设置与作用

| 设置 | 作用 |
| --- | --- |
| `model_provider`、`model` | 选择 CPA 供应商和默认模型 |
| `name`、`base_url`、`wire_api="responses"` | 声明供应商和连接接口 |
| `requires_openai_auth=false` | 使用自定义供应商认证路径 |
| `x-openai-actor-authorization` 请求头 | 当前版本下启用客户端图片执行器的社区兼容标记 |
| `auth.command`、`auth.args` | 从现有 `auth.json` 读取 Bearer 密钥；同时使模型目录进入主动刷新路径 |
| `image_generation=true` | 启用图片功能 |
| `features.code_mode.direct_only_tool_namespaces=["image_gen"]` | 在 Code Mode 下直接暴露图片工具 |

`local-image-extension` 是兼容标记，不是 OpenAI 签发的身份令牌，也不代替 CPA 的 API Key。网关仍然需要有效的 Bearer 认证。不要将这套标记当作通用网关认证方案。

使用命令认证时，不要同时设置 `env_key`、`experimental_bearer_token` 或 `requires_openai_auth=true`。命令的标准输出由 Codex 用作密钥，不要在聊天、截图或日志中直接执行并展示输出。

`supports_websockets`、推理强度是可选项。本机插件、MCP、项目路径、通知脚本、桌面偏好、多代理设置和手动上下文长度均不属于这份复用模板的必要内容。精简供应商模板不意味着应该删除其他电脑原有的个人配置。

## 为什么只开 image_generation 不够

排查本版本源码和实际请求后，发现两个条件：

1. 普通 API Key 认证未满足内置图片执行器的认证分支。
2. 修正认证后，当前模型的 `code_mode_only` 模式下仍缺少直接图片工具；指定 `direct_only_tool_namespaces` 后，出站请求出现 `image_gen.imagegen`。

`imagegenext` 是旧别名，并不是另一个必须开启的开关。也不需要关闭 Responses Lite、替换模型、修改 Codex 二进制或伪造 ChatGPT 登录凭据。

## 为什么 CPA 全部模型也出现了

关键是改用了 `auth.command`，不是手动写入模型清单。

本版本模型管理器将 `has_command_auth()` 作为主动刷新模型目录的条件。配置里的 CPA `base_url` 使它向 CPA 请求 `/v1/models`；认证助手提供同一份现有密钥。实测模型缓存从 8 项变成 44 项，用户在桌面模型选择器中看到了 Claude、Gemini、Grok 等模型。

模型可见表示目录加载成功，不保证每个模型的工具调用、图片输入或其他协议能力都兼容。

## 如何验证，而不是只看模型说“能用”

本次做过以下验证：

- 通过本机 Codex 和本地模拟 Responses 服务检查实际出站工具定义：原配置没有 `image_gen`，修复配置有。
- 确认请求携带认证，认证助手读取结果与已有密钥一致，Codex 和 cc-switch 的目标配置一致。
- 使用模拟 Responses 返回工具调用，触发真实 Codex 内置图片执行器；图片请求转发到真实 CPA，收到 HTTP 200，Codex 保存 PNG，并完成解码与画面检查。
- 后续用户确认 CPA 模型已显示在桌面端。

上述内置执行器测试中的工具调用由模拟服务触发，不等同于验证新桌面会话中的模型自主选择工具。应用配置后仍应在桌面端执行一次实际图片请求，检查原生工具调用和输出图片。

直接用 Python/CLI 调用 `/images/generations` 成功，只能证明 CPA 图片接口可用，不能单独证明 Codex 原生工具恢复。

## 常见问题

- **切换供应商后失效：** 只改了 Codex 文件，没同步 cc-switch 保存的供应商配置。
- **401：** 检查 `auth.json` 的 API Key，以及命令认证是否成功。仅改 `requires_openai_auth=false` 会丢失原来的隐式认证路径。
- **没有 image_gen：** 检查特性开关、直接工具命名空间、认证配置，并重启；新版本可能改变启用条件。
- **模型列表不完整：** 检查 CPA 的 `/v1/models`、认证、网络和缓存刷新。不要公开输出密钥。
- **网络 502：** 本次 Python 诊断受系统代理影响，直连 CPA 后恢复；这是独立网络问题，不是图片工具缺失的根因，不要因此全局关闭代理。

## 参考资料

- [Codex 本版本图片工具启用条件](https://github.com/openai/codex/blob/rust-v0.162.0-alpha.2/codex-rs/core/src/tools/spec_plan.rs)
- [Codex 本版本配置结构](https://github.com/openai/codex/blob/rust-v0.162.0-alpha.2/codex-rs/core/config.schema.json)
- [模型目录刷新条件](https://github.com/openai/codex/blob/rust-v0.162.0-alpha.2/codex-rs/models-manager/src/manager.rs)
- [模型目录请求实现](https://github.com/openai/codex/blob/rust-v0.162.0-alpha.2/codex-rs/model-provider/src/models_endpoint.rs)
- [社区同类兼容方案](https://github.com/Wei-Shaw/sub2api/issues/4253)
- [官方命令认证文档](https://learn.chatgpt.com/docs/config-file/config-advanced#custom-model-providers)
