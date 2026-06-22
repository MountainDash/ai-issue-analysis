# ai-issue-analysis

一个通用的 GitHub composite action，用来在 Issue 打开或被评论时调用 AI CLI（Codex 或 Claude Code）做分析，并把分析过程和最终结论持续回写到同一条评论里。

实战效果展示：

- [Bot 自动分析回复 ISSUE: 加载时间过长导致拜访好友失败](https://github.com/MaaEnd/MaaEnd/issues/1361#issuecomment-4071450863)
- [Bot 响应 @ 进行分析回复 ISSUE: 换班时会把训练室干员换下](https://github.com/MaaAssistantArknights/MaaAssistantArknights/issues/15963#issuecomment-4067281056)

## 快速接入

### Codex（默认）

1. 准备一个 OpenAI 兼容的 API Key（OpenAI 官方、Azure、或其他兼容服务均可）

2. 在你的 GitHub 仓库 - Settings - secrets - actions - new repository secret

     - Name: `CODEX_API_KEY`
     - Secret: 你的 API Key

   如果你使用非 OpenAI 官方端点，还需要额外添加：

     - Name: `CODEX_BASE_URL`
     - Secret: 你的 API base URL（例如 `https://your-proxy.example.com/v1`）

3. 把下面两个文件拷贝到你的仓库里，文件夹不要变

    - [`.github/workflows/ai-issue-analysis.yml`](.github/workflows/ai-issue-analysis.yml)
    - [`.claude/skills/generic-issue-log-analysis/SKILL.md`](.claude/skills/generic-issue-log-analysis/SKILL.md)

4. 新提个 issue 测试下能否正常运行了，或者在以前的 issue 里 `@github-actions`

### Claude Code

1. 准备一个 Anthropic API Key

2. 在你的 GitHub 仓库 - Settings - secrets - actions - new repository secret

     - Name: `ANTHROPIC_API_KEY`
     - Secret: 你的 API Key

3. 在 workflow 中将 `ai-tool` 设为 `claude`，并将 `ai-api-key` 指向 Anthropic 的 key：

    ```yaml
    - uses: MistEO/ai-issue-analysis@codex
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        ai-tool: claude
        ai-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
        ai-model: claude-sonnet-4-20250514
        bot-name: "@github-actions"
    ```

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
- `ai-tool`: AI 工具选择，`codex`（默认）或 `claude`
- `ai-api-key`: AI CLI 使用的 API Key，支持传多个 key，每行一个，action 会随机选择一个使用。如果为空，会 fallback 到 `codex-api-key`
- `ai-base-url`: API 端点 base URL，仅 `ai-tool=codex` 时生效。如果为空，会 fallback 到 `codex-base-url`
- `ai-model`: 模型名称，默认 `gpt-5.5`（codex）或按 claude 默认。如果为空，会 fallback 到 `codex-model`
- `codex-package`: Codex CLI 的 npm 包名，默认 `@openai/codex`，仅 `ai-tool=codex` 时生效
- `codex-api-key`: **[已废弃]** 请改用 `ai-api-key`
- `codex-base-url`: **[已废弃]** 请改用 `ai-base-url`
- `codex-model`: **[已废弃]** 请改用 `ai-model`
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
- `analysis-prompt`: 本次最终传给 AI CLI 的 prompt
- `ai-output`: 完整执行日志，包含启动前参数打印、prompt 正文，以及 AI CLI 输出
- `final-conclusion`: AI CLI 写入 `answer-file` 的最终结论
- `analysis-prompt`、`ai-output` 和 `final-conclusion` 在过长时会为适配 GitHub Actions output 大小限制而被截断；完整内容优先从 artifacts 读取

## 上传产物

- `ai-output-issue-<issue-number>-comment-<comment-id>`: 完整执行日志，包含启动前参数、prompt 正文和 AI CLI 输出
- `final-conclusion-issue-<issue-number>-comment-<comment-id>`: 最终结论文本

## Skill 配合

- 这个 action 只负责 GitHub Actions 编排、评论更新、AI CLI 调用和 prompt 拼接，不内置项目领域知识
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
- 会根据 `ai-tool` 自动安装对应的 CLI（`@openai/codex` 或 `@anthropic-ai/claude-code`）
- 会先创建一条评论，然后持续更新这条评论
- 会导出 `comment-id`、`comment-url`、`analysis-prompt`、`ai-output`、`final-conclusion` 等 action outputs
- `ai-output` 会包含启动前的参数打印和 prompt 正文，不再只是 AI CLI 进程本身的 stdout/stderr
- 会上传 AI CLI 原始输出和最终结论两个 artifacts
- 最终评论会包含最终结论、完整分析过程折叠块，以及当前 Actions 运行链接
- `ai-api-key` 兼容单个 key，也兼容多个 key 按行填写；传多个时每次运行会随机选一个
- 旧的 `codex-api-key`、`codex-base-url`、`codex-model` inputs 仍然可用（作为 deprecated fallback），但推荐迁移到 `ai-api-key`、`ai-base-url`、`ai-model`
