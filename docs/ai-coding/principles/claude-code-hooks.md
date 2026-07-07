---
title: Claude Code Hooks 详解：生命周期钩子与自动化工作流
description: 从 Claude Code 生命周期出发，讲清 Hooks 的触发时机、handler 类型、输入输出、安全拦截、自动格式化和通知提醒，帮助你用 Hooks 把提示词里的软约束变成可审计、可复用的自动化动作。
category: AI 编程原理
tag:
  - Claude Code
  - Hooks
  - AI Agent
  - AI 编程
head:
  - - meta
    - name: keywords
      content: Claude Code,Hooks,生命周期钩子,AI编程,自动化工作流,PreToolUse,PostToolUse,UserPromptSubmit,SessionStart,权限控制
---

用 Claude Code 写代码到一定阶段之后，很多人会遇到同一个问题。

问题通常不在模型能力上。

恰恰相反，是它太能干了。它能改文件、跑命令、查项目结构、生成脚本，也能一口气处理一串很长的任务。于是你会很自然地开始把更多动作交给它。

然后问题就来了。

改完文件，它这次会不会忘了格式化？

准备跑 Bash 命令时，它会不会不小心带上 `rm -rf`？

它会不会顺手改到 `.env`、`.git/` 或生产配置？

它卡在权限弹窗时，我能不能不用一直盯着终端？

上下文压缩之后，那些项目规矩能不能自动补回来？

这些问题有个共同点，它们都不适合只靠提示词解决。

提示词解决的是“尽量记得”。Hooks 解决的是“到了这个时刻，就一定执行”。

这两者的差别，可以先用一张图概括：

![Prompt 提醒依赖上下文和模型记忆，Hooks 卡点通过自动触发、脚本审计和风险阻断保证动作发生](https://oss.javaguide.cn/github/javaguide/ai/coding/claudecode/hooks-vs-prompts-guarantee.webp)

我喜欢把 Hooks 理解成 Claude Code 工作流里的固定卡点：在会话开始、用户提交 Prompt、工具调用前、工具调用后、任务停止前、上下文压缩前后这些生命周期节点上，按配置执行你写好的动作。

这篇文章我不太想写太多，重点帮你搞清楚这些问题：Hooks 到底是什么、解决了什么问题；什么场景改用 Hooks；Hooks 和 Skills 如何选择？

## Hooks 到底是什么

官方文档对 Hooks 的定义是：**Hooks 是用户定义的 shell commands、HTTP endpoints 或 LLM prompts，会在 Claude Code 生命周期的特定点自动执行。**

这句话你只需要抓住两个词就行：**生命周期节点和自动执行** 。

前者决定 Hook 什么时候触发，后者决定它不是靠 Claude 临场想起来，而是按你写好的命令或脚本跑。尤其是 `command` hook，它不依赖模型临场判断，所以更稳定，也更容易审计。

如果把这些触发点摊开，Hooks 更像分布在 Claude Code 生命周期里的固定卡点：

![Claude Code Hooks 围绕 SessionStart、UserPromptSubmit、PreToolUse、PostToolUse、PermissionRequest 和 PreCompact 等生命周期节点自动执行](https://oss.javaguide.cn/github/javaguide/ai/coding/claudecode/claude-code-hooks-lifecycle-map.webp)

Hook handler 主要有五类：

| 类型       | 做什么                               | 适合场景                               |
| ---------- | ------------------------------------ | -------------------------------------- |
| `command`  | 执行 shell command                   | 格式化、日志、安全拦截、通知           |
| `http`     | 把事件 JSON POST 到一个 URL          | 团队审计服务、远程通知、集中化策略     |
| `mcp_tool` | 调用已连接 MCP server 上的工具       | 复用现有 MCP 能力                      |
| `prompt`   | 用一次模型判断返回 yes/no 风格 JSON  | 轻量判断，比如 Stop 前检查任务是否完成 |
| `agent`    | 启动带工具访问能力的 subagent 做验证 | 需要读文件、搜代码、跑命令的验证       |

日常项目里，先把 `command` 当默认选项就行。规则能写成脚本，就别急着让模型判断；脚本更好测，也更容易 review。

`agent` hooks 目前在官方文档里仍标注为 experimental。它能做更复杂的验证，但调试成本也会跟着上来。

我会更倾向于先用 `command`，只有确实需要模型读代码、跑测试、综合判断时，再考虑 `prompt` 或 `agent`。

把这五类 handler 放到一起看，选择顺序会更清楚：

![Hook handler 包括 command、http、mcp_tool、prompt 和 agent，优先使用稳定可审计的 command 脚本](https://oss.javaguide.cn/github/javaguide/ai/coding/claudecode/hook-handler-types.webp)

## Hooks 到底解决了什么问题

Claude Code 确实已经很强，但它不一定每次都在正确时机做同一件事。

比如格式化。

你可以在 `CLAUDE.md` 里写“改完代码请运行 Prettier”。大多数时候它会照做。但上下文长了、任务绕了几圈、中途又插入了新要求，它仍然可能漏掉。

如果项目规则还没整理清楚，可以先看 [CLAUDE.md 最佳实践](https://javaguide.cn/ai-coding/practices/claude-md-best-practices.html)。但要注意，`CLAUDE.md` 更像软约束；能被脚本、Hook、Linter 或 CI 机械化验证的规则，最好不要只停留在自然语言提醒里。

再比如敏感文件保护。

你当然可以告诉 Claude Code “不要改 `.env`”。但这条规则一旦被埋在几十轮对话里，或者某个任务看起来必须改配置，模型就可能把它当成普通建议处理。

这就是 Hooks 该出场的地方。

格式化、危险命令检查、权限通知、压缩后补规则，这些动作不应该靠 Claude 每次自己想起来。

放到工程里看，它和 pre-commit、CI、lint-staged、CODEOWNERS、branch protection 是一类东西：把必须发生的动作从记忆里拿出来，放进流程里。它们存在的原因很简单，再聪明的人也会累、会忘、会手滑。

Claude Code 也是一样。

AI 编程越深入，问题越会从“模型能不能写代码”，转向“谁来保证那些必须发生的动作真的发生”。

Hooks 就是这套保证机制的一部分。

## Hooks 最小配置

Hook 配在 Claude Code 的 settings 文件里。常用位置有三个：

| 位置                          | 作用范围             | 适合放什么                   |
| ----------------------------- | -------------------- | ---------------------------- |
| `~/.claude/settings.json`     | 当前用户所有项目     | 个人通知、个人习惯           |
| `.claude/settings.json`       | 当前项目，可提交仓库 | 团队共享规则、项目级安全限制 |
| `.claude/settings.local.json` | 当前项目本机私有     | 不适合提交的个人配置         |

官方还支持 managed policy、插件的 `hooks/hooks.json`，以及 skill 或 agent frontmatter 里的 hooks。

日常写项目，先记住上面三个就够了。

一个最小配置长这样：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

拆开看，其实就是三层：

- `PostToolUse` 是事件名，表示工具调用成功之后触发。
- `matcher` 是过滤条件。这里写 `Edit|Write`，只在 Claude Code 使用 `Edit` 或 `Write` 改文件之后触发。官方也提到，在较新的 Claude Code 版本里，工具名 matcher 可以用 `|` 或 `,` 分隔列表。
- `hooks` 数组里是真正执行的 handler。这里是一个 `command`，会从 stdin 的 JSON 里取出刚编辑的文件路径，再交给 Prettier。

示例为了短，把命令直接写进了 JSON。实际项目里，只要命令开始变长，或者要引用项目里的脚本，我更建议写成独立文件，再用 `${CLAUDE_PROJECT_DIR}` 指过去：

```json
{
  "type": "command",
  "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/format-after-edit.sh",
  "args": []
}
```

这里的 `args` 不是摆设。官方文档建议，引用项目路径、插件路径这类占位符时优先用 exec form；每个 `args` 元素都会作为一个独立参数传给脚本，不再经过 shell 分词。路径里有空格、括号或特殊字符时，这比自己在一长串 shell command 里补引号稳得多。

如果你省略 `matcher`、写空字符串，或者写成 `.*` 这样的全匹配正则，这个 hook group 会在对应事件的每一次发生时触发。

这听起来省事，但通常不是好事。

格式化 hook 写得太宽，可能每次工具调用后都跑 formatter。权限 hook 写得太宽，可能每个授权弹窗都被自动处理。安全拦截写得太宽，调试起来也很烦。

Hooks 的第一原则就是收窄。

能写 `Edit|Write`，就别写全匹配。

能只拦 `Bash` 里的危险命令，就别让所有工具都进同一个脚本。

## Hook 输入输出怎么工作

Hook 触发时，Claude Code 会把事件上下文作为 JSON 传给 handler。

如果是 `command` hook，这段 JSON 走 stdin。

如果是 `http` hook，这段 JSON 会作为 POST body 发给服务端。

所有事件都会有一些公共字段，比如：

| 字段              | 含义                       |
| ----------------- | -------------------------- |
| `session_id`      | 当前会话 ID                |
| `transcript_path` | 会话 JSONL 文件路径        |
| `cwd`             | 触发 hook 时的工作目录     |
| `permission_mode` | 当前权限模式，部分事件才有 |
| `hook_event_name` | 触发的事件名               |

工具相关事件还会带 `tool_name` 和 `tool_input`。

比如 Claude Code 准备执行 `npm test` 时，`PreToolUse` 可能收到这样的输入：

```json
{
  "session_id": "abc123",
  "cwd": "/Users/example/project",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

所以 Hook 脚本里很常见的一段就是：

```bash
INPUT="$(cat)"
TOOL_NAME="$(echo "$INPUT" | jq -r '.tool_name // empty')"
COMMAND="$(echo "$INPUT" | jq -r '.tool_input.command // empty')"
```

这里建议用 `jq` 解析 JSON，不要自己用 grep 拼字段。

这里别按普通脚本的习惯乱写。

`stdout` 在 `exit 0` 时可能会被 Claude Code 当成结构化输出解析，所以不要往里面塞调试日志。错误原因写 `stderr`。想阻断，大多数事件用 `exit 2`；普通非 0 错误很多时候只是 hook 报错，流程还会继续。

最容易踩的坑是 `exit 1`。

在普通 shell 脚本里，`exit 1` 经常表示失败。但在 Claude Code Hooks 里，如果你想强制拦住一个动作，大多数事件要用 `exit 2`。官方 Reference 明确说，`exit 1` 对多数 hook event 是非阻断错误，流程会继续。

再说 JSON 输出。

如果你想更精细地控制，比如 `PreToolUse` 里返回 `allow`、`deny`、`ask`、`defer`，就要 `exit 0`，然后 stdout 只输出一个 JSON 对象。

例如拒绝一次工具调用：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Database writes are not allowed"
  }
}
```

如果是 `PermissionRequest`，结构又不一样，重点在 `decision.behavior`：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow"
    }
  }
}
```

别把 stdout 当日志打。

如果你要输出 JSON，stdout 就只放 JSON。调试信息写 stderr，或者写到日志文件。否则很容易遇到 `JSON validation failed`，然后盯着配置怀疑人生。

还有一点，JSON 只在 `exit 0` 时处理。如果脚本 `exit 2`，stdout 里的 JSON 会被忽略，Claude Code 会使用 stderr 作为反馈。

把输入、输出和返回码放在一起看，大概是这条决策链：

![Hook 从事件 JSON 获取输入，根据 stdout JSON、stderr 日志以及 exit 0、exit 1、exit 2 决定继续、报错或阻断](https://oss.javaguide.cn/github/javaguide/ai/coding/claudecode/hook-input-output-decision.webp)

同一个事件下，如果有多个 Hook 同时命中，Claude Code 会让它们都跑完再合并结果。一个 Hook 返回 deny，不会阻止旁边那个 Hook 写日志、发 HTTP 请求或改文件；`PreToolUse` 里多个决策合并时，会采用更严格的结果。

所以，只要 Hook 会写日志、发请求、改文件，就应该自己判断要不要执行。不要假设另一个安全 Hook 会先跑、会先拦住风险。

改工具输入也一样要克制。官方文档特别提醒过，**如果多个 Hook 都尝试改同一个工具输入，最后生效的是最后完成的那个；但 Hook 是并行执行的，谁最后完成并不稳定。**

## 常用生命周期事件怎么理解

官方文档里列出的事件不少，从会话、工具、权限、子 agent、任务、配置变化、工作树，到 MCP elicitation 都有。

事件名很多，但刚开始真正常用的就几类：会话开始、用户提交 Prompt、工具执行前、工具执行后、权限弹窗、停止响应、上下文压缩。

| 事件                | 触发时机                          | 适合做什么                             |
| ------------------- | --------------------------------- | -------------------------------------- |
| `SessionStart`      | 会话开始或恢复时                  | 注入动态上下文、加载环境、压缩后补规则 |
| `UserPromptSubmit`  | 用户提交 Prompt 后，Claude 处理前 | Prompt 审计、轻量拦截、补动态上下文    |
| `PreToolUse`        | 工具调用执行前                    | 拦危险命令、保护敏感文件、修改工具输入 |
| `PermissionRequest` | 权限确认框出现时                  | 审计权限，或非常窄地自动批准           |
| `PostToolUse`       | 工具调用成功后                    | 格式化、记录日志、lint、补充上下文     |
| `Notification`      | Claude Code 发送通知时            | 桌面通知、手机推送                     |
| `Stop`              | Claude 完成一轮响应时             | 完成通知、质量门禁、提醒继续处理       |
| `PreCompact`        | 上下文压缩前                      | 备份状态、阻止不合适的压缩             |
| `PostCompact`       | 上下文压缩后                      | 记录摘要、同步外部状态                 |

再往下，是一批进阶事件。知道有它们就行，用到时查官方 Reference或者直接问 AI。

| 类别              | 事件                                                                                     |
| ----------------- | ---------------------------------------------------------------------------------------- |
| 会话和配置        | `Setup`、`InstructionsLoaded`、`ConfigChange`、`CwdChanged`、`FileChanged`、`SessionEnd` |
| 提示词和展示      | `UserPromptExpansion`、`MessageDisplay`、`TeammateIdle`                                  |
| 工具和权限        | `PermissionDenied`、`PostToolUseFailure`、`PostToolBatch`                                |
| 子 agent 和任务   | `SubagentStart`、`SubagentStop`、`TaskCreated`、`TaskCompleted`                          |
| 工作树和 MCP 表单 | `WorktreeCreate`、`WorktreeRemove`、`Elicitation`、`ElicitationResult`                   |
| 停止补充          | `StopFailure`                                                                            |

几个事件需要单独提醒。

`PreToolUse` 是安全拦截的核心，因为它发生在工具真正执行之前。想拦 Bash 命令，想保护 `.env`，想阻止写生产配置，都优先放这里。

`PostToolUse` 发生在工具成功之后，所以它适合收尾，不适合做第一道安全门。比如格式化可以放这里，但敏感文件保护不能只靠它，因为文件已经被改了。它仍然可以用 JSON 给 Claude 提供反馈，或者替换工具输出，只是无法撤销刚刚发生的工具调用。

`PermissionRequest` 可以自动批准或拒绝权限请求。它的触发前提是 Claude Code 准备展示权限对话框，所以脚本化、无头或不同 permission mode 下要按实际会不会出现权限请求来判断。自动化权限最好格外谨慎，别用它全局放行。

这三个事件最容易混，可以先按触发时机和用途这样记：

![PreToolUse 适合在工具执行前拦截风险，PostToolUse 适合工具成功后格式化和记录日志，PermissionRequest 适合权限弹窗时做审计或窄范围批准](https://oss.javaguide.cn/github/javaguide/ai/coding/claudecode/pretooluse-posttooluse-permission.webp)

`Stop` 不等于“任务完成”，它只是 Claude 准备结束本轮响应时触发。如果你用 Stop hook 做质量门禁，要防止循环。官方提供了 `stop_hook_active` 一类字段帮助判断当前是否已经由 Stop hook 继续过。

`PreCompact` 可以阻止压缩，`PostCompact` 不能改变已经完成的压缩结果。压缩后重新注入规则，更常见的做法是用 `SessionStart` 搭配 `compact` matcher。上下文压缩和规则补回属于 Context Engineering 的一部分，想继续展开可以看 [上下文工程实战指南](https://javaguide.cn/ai/agent/context-engineering.html)。

## 三个最小可用示例

真要上手，我建议大家从三个例子开始：一个只负责通知，一个负责改完文件后的收尾，一个放在工具执行前做拦截。

它们刚好覆盖低风险、自动化收益和安全底线三种场景。

### Notification，Claude 需要你时弹个通知

这个适合第一个配，因为它几乎不碰代码，风险最低。

macOS 上可以写到 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

这里 `matcher` 写 `permission_prompt`，表示只有 Claude 需要你批准工具调用时才通知。如果想所有通知都触发，可以省略 matcher 或写空字符串。官方列出的 Notification matcher 还包括 `idle_prompt`、`auth_success`、`elicitation_dialog` 等。

如果 macOS 没弹通知，先在终端手动跑：

```bash
osascript -e 'display notification "test"'
```

然后去系统设置里给 Script Editor 打开通知权限。这个坑很常见，Hook 可能已经触发，只是系统没让通知显示。

### PostToolUse，改完文件自动格式化

前端项目里，最常见的是改完 `Edit` 或 `Write` 后跑 Prettier：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

这段配置有三个关键信息。

`matcher` 只匹配 `Edit|Write`，所以读文件、跑 Bash、调用 MCP 工具都不会触发格式化。

`command` 从 stdin JSON 里拿 `.tool_input.file_path`，再交给 `npx prettier --write`。

这个 Hook 在 `PostToolUse` 上，所以它是“工具执行后收尾”。formatter 失败时，你可以让错误暴露出来，也可以改成脚本，按文件后缀选择不同 formatter。

比如更稳一点的脚本：

```bash
#!/usr/bin/env bash
set -euo pipefail

file="$(jq -r '.tool_input.file_path // empty')"

case "$file" in
  *.js|*.jsx|*.ts|*.tsx|*.json|*.md)
    npx prettier --write "$file"
    ;;
esac
```

Hook 没有魔法。如果你是 Java 项目，应该换成 `spotlessApply`、`google-java-format` 或项目里已有的格式化命令。如果你是 Python 项目，可能是 `ruff format`。先贴着项目现有工具走，不要为了写 Hook 新造一套格式化体系。

### PreToolUse，阻止危险命令和敏感文件

真正的安全拦截要放在 `PreToolUse`。

先写一个脚本，比如 `.claude/hooks/guard.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
tool="$(jq -r '.tool_name // empty' <<<"$input")"
command="$(jq -r '.tool_input.command // empty' <<<"$input")"
file="$(jq -r '.tool_input.file_path // empty' <<<"$input")"

if [[ "$tool" == "Bash" ]] && [[ "$command" =~ rm[[:space:]]+-rf|chmod[[:space:]]+-R[[:space:]]+777 ]]; then
  echo "Blocked risky shell command: $command" >&2
  exit 2
fi

if [[ "$tool" == "Edit" || "$tool" == "Write" ]]; then
  case "$file" in
    *.env|*.env.*|*/.env|*/.git/*|*id_rsa*|*id_ed25519*)
      echo "Blocked sensitive file edit: $file" >&2
      exit 2
      ;;
  esac
fi

exit 0
```

给它执行权限：

```bash
chmod +x .claude/hooks/guard.sh
```

再挂到项目级 `.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/guard.sh"
          }
        ]
      }
    ]
  }
}
```

这个例子的重点不是那几条规则写得多全。

重点是位置和返回。

它在工具执行前检查，所以能真正拦住。命中风险后，脚本把原因写到 stderr，然后 `exit 2`。Claude Code 会阻止这次工具调用，并把原因反馈给 Claude，Claude 通常会换一种做法。

实际项目里，敏感清单要按自己的情况改。生产配置、凭证文件、迁移脚本、锁文件、CI 配置，都可以逐步加进去。

这里别只靠一条命令黑名单兜底。比如只拦 `rm *`，不代表能拦住 `/bin/rm`、`find -delete` 这类变体。高风险操作最好同时结合路径限制、权限配置、Hooks、Sandbox、CI 和人工 Review。

## 非 command Hook 怎么选

前面的示例都用 `command`，不是因为其他类型不重要，而是因为脚本最稳定、最好调试，也最适合放进工程流程。

`http` 适合接团队审计系统、远程通知或集中化策略。服务端返回的 JSON body 会按 command hook 的 JSON 输出格式处理。这里有个容易误会的点：HTTP 状态码本身不负责阻断工具调用；真要做决策，需要返回 2xx，并在 response body 里带上符合 schema 的字段。

`mcp_tool` 适合复用已经接好的 MCP 能力，但它不会触发 OAuth，也不会帮你建立连接。`SessionStart`、`Setup` 这类事件发生得很早，MCP server 可能还没准备好，第一次调用失败并不奇怪。

`prompt` 和 `agent` 都会把判断交给模型。前者适合 Hook 输入本身已经足够判断的轻量场景，比如 Stop 前检查“任务是否真的完成”；后者可以启动 subagent 读文件、搜代码、跑命令，但官方仍标了 experimental。

所以我的选择顺序很简单：规则能写成脚本，就先用 `command`；需要集中审计，再接 `http`；已有稳定 MCP 能力、连接时机也合适，再用 `mcp_tool`；只有判断确实需要模型参与时，才考虑 `prompt` 或 `agent`。

Hooks 是为了把确定的动作固定下来。能不把判断交回模型，就先别交回去。

## Hooks 和 Skills 到底怎么分

这个问题特别容易混。

官方 Skills 文档说，Skills 通过 `SKILL.md` 扩展 Claude 的能力。Claude 会在相关时使用 skill，你也可以用 `/skill-name` 显式调用。Skill 的正文只有在使用时才加载进上下文，所以很适合沉淀长流程、检查清单、项目知识、脚本和参考资料。

如果想系统理解 Skills 和 Prompt、MCP、Function Calling 的分工，可以看 [Agent Skills 是什么？和 Prompt、MCP 到底差在哪？](https://javaguide.cn/ai/agent/skills.html)。

![Agent 执行链路](https://oss.javaguide.cn/github/javaguide/ai/skills/skill-agent-execution-link.webp)

Hooks 则完全不同。

Hooks 不负责给 Claude 一份说明，它负责在生命周期节点上自动执行动作。

可以这样分：

| 维度             | Hooks                                                        | Skills                                                   |
| ---------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| 触发方式         | Claude Code 生命周期事件自动触发                             | Claude 判断相关时加载，或用户手动 `/skill-name`          |
| 核心价值         | 让固定动作稳定发生                                           | 给 Claude 增加某类能力或流程知识                         |
| 适合场景         | 格式化、危险命令拦截、权限审计、通知、日志、质量门禁         | 代码审查流程、部署 SOP、故障排查、资料检索、复杂任务处理 |
| 对模型判断的依赖 | 低，尤其是 `command` hook                                    | 更高，Claude 需要理解并执行 skill 指令                   |
| 是否适合阻断     | 适合，尤其是 `PreToolUse`、`UserPromptSubmit`、`Stop` 等事件 | 不适合作为硬拦截机制                                     |
| 常见风险         | matcher 写太宽、脚本慢、自动批准过度                         | 描述不清、触发不准、流程太长                             |

一句话判断：

如果这件事必须每次发生，放 Hooks。

如果这件事需要 Claude 理解上下文、做选择、按步骤完成，放 Skills。

比如“每次改完 TypeScript 文件都跑 Prettier”，这是 Hooks。

比如“按团队标准做一次 PR review”，这是 Skills。

比如“任何时候都不能改 `.env`”，这是 Hooks。

比如“排查线上接口超时，先看日志，再看指标，再给回滚建议”，这是 Skills。

它们也可以配合。

Skill 教 Claude 怎么做代码审查，Hook 保证它改完文件后格式化、遇到危险命令前被拦、结束前检查有没有测试结果。

一个管能力。

一个管纪律。

放到这个场景里就好理解了。

## 实际落地，先配 2 到 3 个

很多人第一次看到 Hooks，会想把所有生命周期都挂满。

先别急。

Hooks 越多，调试成本越高。你会很快遇到一种问题：Claude 为什么没继续？是 `Stop` hook 拦了？是 `PreToolUse` deny 了？是 `PermissionRequest` 自动处理了？还是某个 `PostToolUse` 脚本超时了？

小 G 会建议从三个高收益 Hook 开始。

第一个，`Notification`。

等待授权、等待输入时提醒你。这个不碰代码，风险低，收益直接。

第二个，`PostToolUse` 自动格式化。

只对你确定有 formatter 的文件类型启用。前端就 Prettier，Python 就 Ruff，Java 就项目现有格式化工具。别全仓库乱跑。

第三个，`PreToolUse` 安全拦截。

先拦最危险的：删除命令、递归提权、`.env`、`.git/`、私钥、生产配置。这些一旦出事，后果比少跑一次格式化严重得多。

再往后，可以考虑：

- 用 `SessionStart` 的 `compact` matcher，在压缩后重新注入关键规则。
- 用 `PreCompact` 在压缩前记录当前任务和摘要。
- 用 `ConfigChange` 审计 Claude Code 配置变化。
- 用 `CwdChanged` / `FileChanged` 配合 `CLAUDE_ENV_FILE` 重新加载环境。
- 用 `Stop` 做完成通知或轻量质量门禁。

权限自动批准要单独拎出来说。

`PermissionRequest` 确实能自动批准权限请求，官方示例里就自动批准了 `ExitPlanMode`。但这个能力很锋利。

matcher 要窄。

输入要检查。

能继续保留人工确认的，就保留。

尤其是删除、生产环境、凭证文件、外部 API 写操作，不要为了少点几次确认把门锁拆了。

## 常见问题

**Hook 会消耗很多 token 吗？**

普通 `command` Hook 不会让模型参与，成本主要是本机命令耗时、外部服务调用和脚本自身副作用。`prompt` 和 `agent` Hook 会用模型，才需要考虑 token、超时和返回不稳定。

**stdout 写什么都会进 Claude 上下文吗？**

不会。`UserPromptSubmit`、`UserPromptExpansion`、`SessionStart` 这类事件的 stdout 更容易被当成 Claude 可见上下文处理；多数事件里，stdout 主要用于 JSON 输出或结构化决策。要返回 JSON 时，stdout 里就只放一个 JSON 对象，调试日志写 stderr 或文件。

**能不能用 Hook 触发 slash command 或工具调用？**

`command` Hook 只能通过 stdout、stderr 和 exit code 和 Claude Code 通信，不能直接触发 `/` commands 或 tool calls。要调用 MCP 工具，用 `type: "mcp_tool"`；要让模型参与判断，用 `prompt` 或 `agent`。

**为什么我的 Hook 没生效？**

先跑 `/hooks` 看它有没有注册到正确事件。`/hooks` 是只读浏览器，用来看配置来源、事件、matcher、handler 类型、命令或 URL，它不负责编辑配置。

然后检查配置文件位置和 JSON。用户级是 `~/.claude/settings.json`，项目共享是 `.claude/settings.json`，本机私有是 `.claude/settings.local.json`。再看 matcher，它不总是匹配工具名：`Notification` 匹配通知类型，`SessionStart` 匹配启动来源，`PreCompact` 和 `PostCompact` 匹配 `manual` 或 `auto`。

还有一种情况容易忽略：`PermissionRequest` 依赖权限确认流程，非交互模式下未必会出现权限弹窗。这类自动化如果要稳定拦截，通常应该优先放到 `PreToolUse`。

**想阻断工具调用，用 `exit 1` 行吗？**

大多数情况下不行。想拦住动作，通常要用 `exit 2`，或者 `exit 0` 后输出符合要求的结构化 JSON。普通非 0 错误很多时候只会显示 hook error，然后流程继续。

**`PostToolUse` 能不能做安全门？**

不适合。它发生在工具执行之后，已经晚了。保护敏感文件、拦危险命令，要用 `PreToolUse`。`PostToolUse` 更适合格式化、记录日志、补充上下文或把工具结果整理后反馈给 Claude。

**Hook 和权限规则冲突时谁更硬？**

`PreToolUse` 发生在权限检查前，可以把风险动作提前拦下来。更适合把它理解成“加严”机制：Hook 返回 deny 可以挡住危险调用，但 Hook 返回 allow 不能绕过 settings 里的 deny 规则。项目里的 deny、权限模式、沙箱和人工确认，仍然应该按最高风险来设计。

## 小结

Hooks 最适合处理那些“时机固定、动作明确、最好能记录或阻断”的事情。比如改完文件格式化、执行前拦危险命令、保护 `.env` 和私钥、等待权限时通知你、压缩前后记录状态，这些都属于 Hooks 的舒适区。

反过来看，如果一件事需要 Claude 读上下文、理解任务目标、自己选择执行路径，那就不要硬塞进 Hook。它更适合放进 Skill，或者留在当前任务里让 Claude 判断。Hooks 管固定时刻的动作，Skills 管可复用的做事方法。

起步也不用复杂。先配一个通知、一个格式化、一个安全拦截，把这三个跑稳，你就能明显感觉到 Claude Code 不再只是一个聪明的聊天框，而是开始有了一点“开发系统”的样子。

我觉得 Hooks 最有意思的地方也在这儿。它不是给 AI 编程再加一层魔法，而是把那些本来就该稳定发生的动作，放回工程流程里。

如果想继续补 Agent、Context Engineering、MCP、Skills 和 AI 编程实践，可以从 [AIGuide：AI 应用开发、AI 编程实战与面试指南](https://mp.weixin.qq.com/s/le3RzJsaAH22auUoB05y1Q) 开始。
