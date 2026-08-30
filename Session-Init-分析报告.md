# TencentDB Agent Memory Session Init 调研报告

> 调研日期：2026-08-26（2026-08-30 对照源码复核并更新）  
> 项目目录：`tencentdb-agent-memory-dsh`  
> 调研任务：实际启动 Session Init，并调研工程实现上可以优化的地方

## 1. 本次实操环境

| 项目 | 本次使用的配置 |
| --- | --- |
| 本机环境 | macOS 26.6.2（Apple Silicon），Docker 29.4.0 |
| Agent 客户端 | Claude Code 2.1.247 |
| 本地服务 | MemoryCore、MemoryHub、MemoryProxy |
| 模型 API | Agnes API：`https://apihub.agnes-ai.com/v1`，经 CC-Switch 转发 |
| API 格式与模型 | OpenAI Chat Completions，`agnes-2.0-flash` |
| 测试资产 | `default-team`、专用调研 Agent、工程优化调研 Task |

本次实际调用链为：

```text
Claude Code → MemoryProxy → CC-Switch → Agnes
```

API Key 已配置，但没有写入本报告。

## 2. 调研过程

### 2.1 启动环境

Claude Code 通过 Homebrew 安装时出现下载中断，之后改用官方安装脚本完成安装。CC-Switch 中新增 Agnes Provider，并将 Claude 的 Sonnet、Opus、Haiku 映射到 `agnes-2.0-flash`。

项目 rebase 到 `origin/feat/server_team` 后，启动本地服务：

```bash
cd tencentdb-agent-memory-dsh/deploy/global-images
./start-all.sh
```

MemoryCore、MemoryHub、MemoryProxy 均启动成功。启动过程中还修复了两个 shell 脚本在中文标点附近的变量解析问题。

### 2.2 启动 Claude Code

为了确保请求先经过 MemoryProxy，而不是被全局 CC-Switch 配置直接接管，本次使用以下命令：

```bash
cd tencentdb-agent-memory-dsh
claude \
  --setting-sources project,local \
  --settings ../notes/claude-memory-proxy.settings.json \
  --model agnes-2.0-flash
```

启动后发送第一条正式消息，MemoryProxy 会自动触发 Session Init，不需要另外告诉 Agent“执行 Session Init”。由于本次没有通过 Header 预设资产，界面会依次询问 Team、Agent 和 Task。

### 2.3 两次会话测试

第一次会话关联了 Team 和专用 Agent，并让 Agent 调研 TencentDB Agent Memory 的工程优化方向。Task 选择时操作过快，实际选中了“不关联任务”。

第二次新建 Claude Code 会话，重新选择同一个 Team、Agent 和真实 Task，用于确认 Session Init 是否成功，以及能否取回上一会话的调研结论。

之后又通过 MemoryProxy 补测了 Header 自动初始化、非法资产、主动 bypass、同会话续聊和 Proxy 重启恢复等场景。

## 3. 实测结果

| 测试案例 | 结果 | 说明 |
| --- | --- | --- |
| Claude Code → MemoryProxy → CC-Switch → Agnes | 通过 | Agnes 模型正常回复 |
| 交互式 Session Init | 通过 | Team、Agent、真实 Task 可以正常关联 |
| 完整 Team/Agent/Task Header | 通过 | 不弹表单，直接完成注册 |
| 已绑定会话继续请求 | 通过 | 不再要求选择资产 |
| 已绑定会话跨 Proxy 重启 | 通过 | 从 L2a 持久层恢复原绑定 |
| 主动选择“不关联资产” | 通过 | 当前会话 bypass，并跳过全部注入 |
| bypass 会话跨 Proxy 重启 | 通过 | 重启后仍保持 bypass 状态 |
| 非法 Team/Agent Header | 通过 | 不信任传入值，回退资产选择表单 |
| 错误 API Key | 通过 | 返回 HTTP 401，未进入 Session Init |
| 缺少或传入过期 Task | 版本存在差异 | 当前镜像回退表单，当前源码设计为 Task 可选（且 workbuddy 通道仍按老逻辑校验，见 4.6） |
| 新会话召回上一会话调研结果 | 未通过 | Memory 工具调用方式与模型行为不匹配 |

MemoryProxy 当前可发现的单测为 2 个文件、9 个用例，本轮重新执行全部通过。但这些单测尚未覆盖 Header 预选、bypass、过期 Task 和重启恢复。

## 4. 调研发现

### 4.1 Session Init 成功不代表记忆召回成功

第二次会话的 Team、Agent、Task 和 Profile/Memory 提示都已注入，但模型把 `tdai_memory_search` 当成 Claude 原生工具调用，最终提示工具不存在。当前提示实际要求模型通过 Bash + curl 检索，因此上一会话的调研结论没有成功取回。

第二次 Claude 会话 transcript 中的实际结果：

```text
assistant tool_use: tdai_memory_search
tool_result: Error: No such tool available: tdai_memory_search

assistant tool_use: tdai_conversation_search
tool_result: Error: No such tool available: tdai_conversation_search
```

同时，MemoryProxy 日志确认注入本身已经执行：

```text
[hook-cache] hook=tdai-memory-tools-injector hit blocks=1
[hook-cache] hook=tdai-profile-memory-injector hit blocks=1
```

### 4.2 虚拟 Task 的处理不一致

第一次会话选择“不关联任务”后，系统使用虚拟 Task ID `default`。查询阶段知道它不是真实 Task，但参与日志阶段仍然查询该 ID，产生 `404 task_not_found`。不同阶段对虚拟 Task 的判断没有完全统一。

MemoryProxy 的实际运行日志：

```text
[session-init:cc] → initialized agent=agt-p2ndw9xjd8 task=default
[session-init:cc] participation-log append failed:
HTTP 404: task_not_found: task not found: default
```

08-30 复核源码，这个问题在 claude-code 和 codebuddy 两条链路上已经修掉：参与日志的写入加了 `regData.task_id` 守卫（`claude-code/init.ts:502`、`codebuddy/init.ts:443`），虚拟 Task 不会再触发这次调用。上面日志是 08-25 镜像的行为，这也再次说明镜像和源码脱节带来的困惑——见 4.3。

### 4.3 运行镜像与当前源码存在差异

本次运行的 `agentmemory/memory-proxy:latest` 镜像构建于 2026-08-25，要求 Team、Agent、Task 三者齐全。当前工作树已经将 Task 改为可选维度，过期 Task 会被丢弃并进入 Agent 范围宽召回。

这说明使用 `latest` 时很难确认线上行为对应哪一版源码，也会干扰问题排查。

镜像检查结果：

```text
image=agentmemory/memory-proxy:latest
image_id=sha256:c8de30142787a5df7937c02c167f2ee37f00505b79036357653a6ce78a29fba5
created=2026-08-25T07:07:51Z
```

同一段 Header 预选逻辑的对比：

```text
运行镜像：res.canRegister = !!res.agentId && !!res.taskId && !res.hadMismatch
当前源码：res.canRegister = !!res.agentId && !res.hadMismatch
```

08-30 再查，镜像还是同一个 `c8de3014`，没有更新过。而这两天主干上已经合入了两个相关改动：#1131 把 Task 改为可选维度（就是上面源码那一行），#1129 让 Header 预选的会话在缓存未命中时也能拿到记忆，不再静默 bypass。后者意味着本报告“非法 Header 回退表单”那条实测结果在源码上的语义已经变了，等新镜像出来需要重测。

### 4.6 Task 可选改造没有覆盖所有通道

Task 可选目前只在 claude-code 和 codebuddy 两条链路生效。workbuddy 的 Header 预选逻辑还是老的三件套校验：team 齐了但缺 agent 或 task，直接 `bypass("incomplete-header")`（`workbuddy/init.ts:297`），不会走“丢弃 Task、Agent 范围宽召回”的新路径。codex、dsh、opencode 没有 Header 预选，暂不受影响。

如果后面做 Task 判断的统一封装，workbuddy 这个分支是最容易漏掉的。

### 4.4 启动配置不够透明

Claude Code 的全局 user settings 会覆盖单独指定的 MemoryProxy 地址。请求虽然能够正常返回，但实际上可能绕过 Session Init。当前必须显式使用 `--setting-sources project,local` 才能确保路由正确。

两个配置文件的实际值不同：

```text
global_user_settings=http://127.0.0.1:15721
memoryproxy_settings=http://127.0.0.1:8096/claude-code/default
```

不排除 user settings 时，首次探测请求没有出现在 MemoryProxy 日志中；添加 `--setting-sources project,local` 后，日志出现：

```text
[injection-debug] agentSource=claude-code sessionInitEnabled=true
[session-init:cc] state=none → uninitialized
```

08-30 复核时发现 `claude-memory-proxy.settings.json` 已经被精简成只剩 `ANTHROPIC_BASE_URL` 一项，没有 AUTH_TOKEN。而当前 Proxy 配置里 `auth.enabled: true`，没有 token 的请求按理应该被 401 拦截。当时能跑通的具体原因还没查清，下次复测前需要先确认认证链路，否则测试结果不可信。

### 4.5 工程基线仍需完善

全量 TypeScript typecheck 目前仍有一批类型和依赖错误，而现有单测数量较少。实际执行结果：

```text
src/anthropicHandler.ts: error TS2322: RequestKind is not assignable to CcRequestKind
src/anthropicHandler.ts: error TS2339: Property 'resetFlow' does not exist
src/codexHandler.ts: error TS2339: Property 'resetFlow' does not exist

Test Files  2 passed (2)
Tests       9 passed (9)
```

08-30 复核，这些错误都还在，而且不止这三条：同一批还有 6 处 `TS2352`（`SessionInfo` 转 `Record<string, string>` 缺索引签名，分布在 anthropicHandler 和 codexHandler）。另外 `resetFlow` 报错是因为 `SessionInitResult` 类型上没这个字段——`SessionInitState` 上其实有（`session/types.ts:120`），两个类型没收拢。全部加起来 9 条左右，修复面比当初看到的大一圈。

密钥传递问题也可以直接从启动脚本中确认：

```text
deploy/global-images/start-memory-hub.sh:106
-e LLM_API_KEY="$MEMORY_LLM_API_KEY"
```

该写法会把敏感值作为 `docker run` 参数传入，建议改为 env-file 或 Secret。

## 5. 优化方向

结合这次启动和测试过程，我认为可以优先处理以下几项：

1. **把 Memory 检索做成真实工具。** 提供正式的 MCP/Claude Tool 定义，不再依赖模型阅读提示后自行拼 curl；同时增加跨会话写入和检索的端到端测试。
2. **统一 Task 判断。** 把“真实 Task、虚拟 Task、过期 Task”的判断封装成统一逻辑，在注册、上下文注入和参与日志中复用。注意 Task 可选目前只落在 claude-code 和 codebuddy 两条链路，workbuddy 还是老校验（见 4.6），封装时要把各通道一起收进来。
3. **固定部署版本。** 镜像使用 commit SHA 或版本号，不直接依赖 `latest`；启动日志中打印源码版本、构建时间和关键配置。
4. **显示最终生效路由。** 启动时直接展示 settings source、Base URL 和模型，避免请求静默绕过 MemoryProxy。
5. **补充自动化案例。** 优先覆盖 Header 校验、无 Task、过期 Task、主动 bypass、重启恢复和跨会话记忆。
6. **补齐工程与安全基线。** 修复 typecheck，改进容器内健康检查，并使用 env-file 或 Secret 传递敏感配置。

## 6. 结论

本次已经跑通 Claude Code、MemoryProxy、CC-Switch 和 Agnes 的完整调用链。Session Init 可以自动触发，交互式绑定、Header 自动绑定、bypass 和跨重启状态恢复都能够工作。

目前最明显的问题不是 Session Init 无法启动，而是初始化完成后的记忆工具没有真正形成稳定调用闭环，同时运行镜像与当前源码存在语义差异——这个差距在 08-30 复核时进一步拉大了（主干合入了 Task 可选和 Header 缓存未命中两个改动，镜像没动）。后续优先解决工具协议、Task 统一判断、版本可追溯和自动化测试，能够比较直接地提升系统的稳定性和可维护性。

## 相关记录

- [`Session Init 完整实测记录`](./notes/Claude-Code-Agnes-Session-Init-实测记录.md)
