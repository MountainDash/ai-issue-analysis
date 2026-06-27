# ai-issue-analysis

> 本项目基于 [MistEO/ai-issue-analysis](https://github.com/MistEO/ai-issue-analysis) 改进而来。基于项目需求进行了调整。

一个通用的 GitHub composite action，用来在 Issue 打开或被评论时调用 [MiMo Code CLI](https://github.com/XiaomiMiMo/MiMo-Code) 做分析，并把分析过程和最终结论持续回写到同一条评论里。默认使用 MiMo Auto 免费通道，零配置即可开始；也支持接入任意 OpenAI 兼容 API。

## 快速接入

### 方式一：MiMo Auto 免费通道（零配置）

1. 把下面两个文件拷贝到你的仓库里，文件夹不要变

    - [`.github/workflows/ai-issue-analysis.yml`](.github/workflows/ai-issue-analysis.yml)
    - [`.claude/skills/generic-issue-log-analysis/SKILL.md`](.claude/skills/generic-issue-log-analysis/SKILL.md)

2. 新提个 issue 测试下能否正常运行了，或者在以前的 issue 里 `@github-actions`

无需配置任何 API Key，MiMo Auto 内置免费通道会自动处理。

### 方式二：自定义 API Provider

如果你需要使用自己的 API Key（OpenAI 官方、Azure、或其他兼容服务），额外配置 secrets：

1. 在你的 GitHub 仓库 - Settings - secrets - actions - new repository secret

     - Name: `MIMO_API_KEY`
     - Secret: 你的 API Key

   如果你使用非 OpenAI 官方端点，还需要额外添加：

     - Name: `MIMO_BASE_URL`
     - Secret: 你的 API base URL（例如 `https://your-proxy.example.com/v1`）

   如果你想指定具体模型（`provider/model` 格式），还可以添加：

     - Name: `MIMO_MODEL`
     - Secret: 模型名称（例如 `openai/gpt-5.5`、`anthropic/claude-sonnet-4-5`）

2. 同上，拷贝两个文件到你的仓库

#### 自定义供应商配置示例

以接入智谱 GLM 为例，workflow 中配置如下：

```yaml
- name: Analyze issue with MiMo Code
  id: mimo
  continue-on-error: true
  uses: MistEO/ai-issue-analysis@codex
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    mimo-api-key: ${{ secrets.MIMO_API_KEY }}
    mimo-base-url: ${{ secrets.MIMO_BASE_URL }}
    mimo-model: "zhipu/glm-5.2" # secrets.MIMO_MODEL 的某个实际值
    bot-name: "@github-actions"
```

> [!NOTE]
> `mimo-model` 必须为 `provider/model` 格式。`/` 前的部分会被用作 provider ID（可随意命名，如 `zhipu`、`custom` 等），`/` 后的部分是实际模型名。
>
> 此示例会生成如下 MiMo Code 配置：
> ```json
> {
>   "provider": {
>     "zhipu": {
>       "npm": "@ai-sdk/openai-compatible",
>       "name": "Custom Provider",
>       "options": {
>         "baseURL": "https://your-api-endpoint.example.com/v1",
>         "apiKey": "<your-key>"
>       },
>       "models": { "glm-5.2": {} }
>     }
>   },
>   "model": "zhipu/glm-5.2"
> }
> ```

> [!TIP]
>
> 如果你的项目有固定的日志包命名、关键日志路径、附件目录、模块映射或上游依赖，建议在这个通用版基础上微调 `SKILL.md`，分析质量会更高。最佳实践参考：
> - [MaaEnd](https://github.com/MaaEnd/MaaEnd/blob/v2/.claude/skills/maaend-issue-log-analysis/SKILL.md)
> - [MaaAssistantArknights](https://github.com/MaaAssistantArknights/MaaAssistantArknights/blob/dev-v2/.claude/skills/maa-issue-log-analysis/SKILL.md)

## 输入说明

- `issue-number`: Issue 编号，通常可以不传：

    - `issues` / `issue_comment` 事件会自动读取 `github.event.issue.number`
    - `workflow_dispatch` 会自动读取输入名为 `issue_number` 的 dispatch 参数
    
    如果你的 workflow_dispatch 输入名不是 `issue_number`，或者你在其他事件里调用这个 action，就显式传 `issue-number`。

- `github-token`: 用于创建和更新 Issue 评论
- `mimo-api-key`: MiMo Code CLI 使用的 API Key，支持传多个 key，每行一个，action 会随机选择一个使用。**留空则使用 MiMo Auto 免费通道**（默认为空）
- `mimo-base-url`: API 端点 base URL，用于配置自定义 provider。留空则使用默认端点或 MiMo Auto
- `mimo-model`: 模型名称，格式为 `provider/model`（如 `openai/gpt-5.5`）。留空则使用 MiMo Auto 默认模型
- `mimo-package`: 安装的 npm 包名，默认 `@mimo-ai/cli`
- `bot-name`: 从 `issue_comment` 正文中剥离掉的 bot mention，比如 `@YourBot`
- `initial-comment-body`: 开始分析时先发出的评论正文
- `action-link-text`: 评论里展示的运行链接文字
- `details-summary`: 分析过程折叠块的标题
- `prompt-template`: 基础分析提示词模板
- `comment-prompt-template`: 有评论补充要求时追加的提示词模板
- `stream-update-interval-seconds`: 流式更新评论的间隔秒数，默认 `30`
- `checkout-repository`: 是否在 action 内部自动执行 `actions/checkout`，默认 `true`
- `answer-file`: AI 写入最终结论的文件路径，默认 `answer.md`
- `extra-comment-content`: 始终追加在每次评论最末尾的额外内容，默认为空

## 输出说明

- `issue-number`: 本次运行实际解析出的 Issue 编号
- `comment-id`: 创建并持续更新的评论 ID
- `comment-url`: 创建并持续更新的评论 URL
- `analysis-prompt`: 本次最终传给 MiMo Code 的 prompt
- `codex-output`: 完整执行日志，包含 MiMo Code 启动前的参数打印、prompt 正文，以及 MiMo Code CLI 输出
- `final-conclusion`: MiMo Code 写入 `answer-file` 的最终结论
- `analysis-prompt`、`codex-output` 和 `final-conclusion` 在过长时会为适配 GitHub Actions output 大小限制而被截断；完整内容优先从 artifacts 读取

## 上传产物

- `codex-output-issue-<issue-number>-comment-<comment-id>`: 完整执行日志，包含启动前参数、prompt 正文和 MiMo Code CLI 输出
- `final-conclusion-issue-<issue-number>-comment-<comment-id>`: 最终结论文本

## Skill 配合

- 这个 action 只负责 GitHub Actions 编排、评论更新、MiMo Code CLI 调用和 prompt 拼接，不内置项目领域知识
- 对需要分析 issue 附件、日志包、运行时配置、跨仓库代码路径的项目，建议配套提供项目自己的 issue 分析 skill
- 一个可行的 skill 一般至少会覆盖这些步骤：读取 issue 正文和评论、定位并下载日志附件、先建立时间线再筛证据、最后回溯到代码和文档做归因
- 如果没有这层 skill，action 仍然能运行，但对日志包、截图、跨模块调用链这类问题，分析质量通常会明显下降
- 最佳实践参考，MaaEnd: `https://github.com/MaaEnd/MaaEnd/blob/v2/.claude/skills/maaend-issue-log-analysis/SKILL.md`
- 最佳实践参考，MaaAssistantArknights: `https://github.com/MaaAssistantArknights/MaaAssistantArknights/blob/dev-v2/.claude/skills/maa-issue-log-analysis/SKILL.md`

## 模板变量：

- `{{issue_number}}`
- `{{answer_file}}`
- `{{comment_body}}`
- `{{repository}}`
- `{{event_name}}`

## 行为说明：

- action 内部会自动 `checkout` 调用方仓库
- 如果调用方已经自己 checkout，或者前置步骤会生成工作区文件，可以把 `checkout-repository` 设为 `false`
- 会自动安装 `@mimo-ai/cli`（MiMo Code CLI）
- 默认使用 MiMo Auto 免费通道，无需配置 API Key；如需自定义 provider，传入 `mimo-api-key` 和可选的 `mimo-base-url`
- MiMo Code 原生支持读取 `.claude/skills/` 目录下的 skill 文件，无需额外符号链接
- 会先创建一条评论，然后持续更新这条评论
- 会导出 `comment-id`、`comment-url`、`analysis-prompt`、`codex-output`、`final-conclusion` 等 action outputs
- `codex-output` 会包含 MiMo Code 启动前的参数打印和 prompt 正文，不再只是 CLI 进程本身的 stdout/stderr
- 会上传 MiMo Code 原始输出和最终结论两个 artifacts
- 最终评论会包含最终结论、完整分析过程折叠块，以及当前 Actions 运行链接
- `mimo-api-key` 兼容单个 key，也兼容多个 key 按行填写；传多个时每次运行会随机选一个
