# Claude Code + Agnes + TencentDB Agent Memory 实测记录

> 实测日期：2026-08-28  
> 工作目录：项目集合目录  
> 安全说明：本文所有 API Key、管理密钥均已脱敏，未写入文档。

## 1. 实测结论

本次完成了以下工作：

1. 安装 Claude Code，绕过 Homebrew 下载中断问题；安装后由 `2.1.231` 自动更新到 `2.1.247`。
2. 在 CC-Switch 中新增 Agnes Provider，并验证 Agnes `/models`、Chat Completions、CC-Switch 本地 Claude 路由均可用。
3. 启动 TencentDB Agent Memory 的 MemoryCore、MemoryHub 和 MemoryProxy。
4. 建立完整调用链：

   ```text
   Claude Code
       -> MemoryProxy（127.0.0.1:8096）
       -> CC-Switch（127.0.0.1:15721）
       -> Agnes API（agnes-2.0-flash）
   ```

5. 在两个独立 Claude Code 交互会话中执行 Session Init：
   - 第一次完成 Agent 关联并调研 TencentDB Agent Memory 的工程优化点。
   - 第二次成功关联同一 Team、Agent 和真实 Task，用于检查初始化与跨会话记忆。
6. Session Init 本身成功；Profile/Memory 注入日志存在，但第二个会话没有拿到第一个会话的六条调研结论，因此“即时跨会话结论召回”尚未验证成功。

## 2. 安装 Claude Code

### 2.1 Homebrew 失败原因

Homebrew Cask 下载约 295 MB 的二进制时出现：

```text
curl: (18) Transferred a partial file
```

这是下载连接提前中断，不是 macOS 架构或 Cask 配置错误。

### 2.2 实际采用的安装方式

改用 Claude Code 官方原生安装器：

```bash
curl --retry 5 --retry-all-errors -fsSL https://claude.ai/install.sh | bash -s stable
```

安装路径：

```text
~/.local/bin/claude
```

`claude doctor` 检查通过。后续 Claude Code 自动更新到 `2.1.247`。

## 3. Agnes 与 CC-Switch 配置

CC-Switch 实测版本为 `3.20.0`。已创建 `agnes-ai` Provider，主要配置如下：

| 配置项 | 值 |
|---|---|
| Base URL | `https://apihub.agnes-ai.com/v1` |
| API 格式 | `OpenAI Chat Completions` |
| Sonnet / Opus / Haiku 映射 | `agnes-2.0-flash` |
| CC-Switch Local Route | `127.0.0.1:15721` |
| API Key | 已配置，本文不记录明文 |

兼容性参数：

```json
{
  "allowed_openai_params": ["thinking", "context_management"],
  "litellm_settings": {
    "drop_params": true
  }
}
```

配置数据库修改前已备份：

```text
~/.cc-switch/backups/pre-agnes-20260827-234617.db
```

验证结果：

- Agnes `/models`：HTTP 200。
- Agnes Chat Completions：正常返回。
- CC-Switch `/v1/messages`：正常返回。
- CC-Switch Provider 已启用，本地 Claude 路由已启用。

## 4. TencentDB Agent Memory 启动与路由

项目目录：

```text
tencentdb-agent-memory-dsh
```

项目已 rebase 到 `origin/feat/server_team`，当前业务提交为 `33a7dff`，相对上游仅领先 1 个提交。冲突解决后保留了上游 snapshot trim 行为和新引入的 `PresetIdentity`。

启动后服务状态：

| 服务 | 端口 | 状态 |
|---|---:|---|
| MemoryCore | `8420` | healthy |
| MemoryHub Panel | `8125` | healthy |
| MemoryHub Knowledge | `8424` | healthy |
| MemoryProxy | `8096` | healthy |
| CC-Switch Local Route | `15721` | 可访问 |

为避免在文件中保存管理密钥，Claude Code 使用 `apiKeyHelper` 从项目生成的 `.admin-key` 中读取凭据。安全配置文件：

```text
notes/claude-memory-proxy.settings.json
```

通过 Agent Memory 启动 Claude Code 的命令：

```bash
cd tencentdb-agent-memory-dsh
claude \
  --setting-sources project,local \
  --settings ../notes/claude-memory-proxy.settings.json \
  --model agnes-2.0-flash
```

这里的 `--setting-sources project,local` 不能省略。Claude Code 的全局用户设置已由 CC-Switch 管理；如果继续加载 user settings，它会覆盖命令指定的 MemoryProxy Base URL，导致请求直接进入 CC-Switch，绕过 Session Init。

另一个限制是：Session Init 必须在交互模式运行。`claude -p` 会禁用 `AskUserQuestion`，从而跳过待确认的资产关联流程。

## 5. Session Init 资产

本次在本地 MemoryCore 创建并使用以下资产：

| 类型 | 名称 | ID |
|---|---|---|
| Team | `default-team` | `team-p08aprntxh` |
| Agent | `TencentDB Agent Memory 工程优化调研 Agent` | `agt-p2ndw9xjd8` |
| Task | `调研 TencentDB Agent Memory 工程优化点` | `task-p2nd1jlohu` |

## 6. 第一次正式会话：初始化与工程调研

会话 ID：

```text
af9fdd5a-4d02-4ceb-b357-36b6d13e4a7b
```

初始化选择：

1. 确认关联资产。
2. 自动选择唯一 Team：`default-team`。
3. 选择专用调研 Agent。
4. Task 菜单操作时方向键和回车发送过快，实际选中了“本次不关联任务”。

日志确认：

- Agent 初始化成功。
- Team 初始化成功。
- Agent Context 注入成功；由于选择的是虚拟 Task `default`，没有生成真实 Task Context。
- Profile/Memory hook 被注入。
- Task 使用了虚拟 ID `default`，没有关联真实 Task。

随后让 Agent 只读调研项目。第一次长回答发生流式解码错误，缩短输出后成功得到六类建议：

1. 合并 Claude Code 与 CodeBuddy 重复的 Session Init 状态机。
2. 拆分过大的 SQLite 数据访问模块。
3. 管理和拆分大体量生成类型/Schema。
4. 拆分 Anthropic Handler 与通用 Handler 的职责。
5. 增强测试与关键链路回归覆盖。
6. 明确顶层入口与资产导入的所有权边界。

Agent 原回答中的“4309 个 TS 文件、8 个测试文件”不准确。重新使用文件列表验证后，仓库约有 `662` 个 TypeScript 文件、`9` 个测试命名文件；这只能说明测试文件数量偏少，不能替代真实覆盖率统计。

### 第一次会话暴露的真实缺陷

当 Session Init 选择“不关联任务”时，代码把 `default` 视为虚拟 Task，查询阶段会跳过真实 Task 获取；但写参与日志时只检查 `task_id` 是否非空，没有排除虚拟 ID。最终请求 `default` Task 并返回 `404 task_not_found`。

相关位置：

```text
MemoryProxy/src/session/claude-code/init.ts
```

建议把“是否为真实 Task”的判断收敛成统一谓词，并在查询、上下文生成、参与日志三处复用。

## 7. 第二次正式会话：再次初始化与记忆验证

会话 ID：

```text
7fa8eedf-1b44-4ff5-92d1-74e939d1f990
```

初始化选择：

1. 确认关联资产。
2. 选择 `default-team`。
3. 选择同一专用调研 Agent。
4. 选择真实 Task：`调研 TencentDB Agent Memory 工程优化点`。

日志确认：

- Team、Agent、真实 Task 三者均初始化成功。
- Agent Context 与 Task Context 均成功生成。
- Profile/Memory hook 与 TDAI Memory Tools hook 均已注入。

但是，模型把提示中的 `tdai_memory_search` 和 `tdai_conversation_search` 当成 Claude 原生工具直接调用，收到 `No such tool available`。注入提示实际要求模型通过 Bash + curl 调用这些能力。强制它直接总结当前上下文后，它确认没有看到上一会话的六条调研结论。

因此第二次验证应精确表述为：

- Session Init：成功。
- Team/Agent/Task 关联：成功。
- Profile/Memory 工具说明注入：成功。
- 上一会话调研结论的跨会话召回：未成功验证。
- 主要阻塞：模型把“逻辑工具名”误判成原生 tool call，没有按注入的 Bash + curl 协议执行。

## 8. 工程优化建议（按优先级）

### P0：先保证正确性与安全

1. 修复虚拟 Task `default` 的参与日志 404。
2. 给 Memory 工具提供真实 MCP/Claude Tool 定义，或将提示改成不易触发原生 tool call 的命令式 curl 工作流；增加跨会话写入—检索端到端测试。
3. 建立 TypeScript typecheck 的绿色基线。当前全量 typecheck 存在 `RequestKind`、`SessionInfo`、`resetFlow`、`traceId`、`agent_source` 和缺失 `@context-proxy/cost-guard` 等错误。
4. 避免用 `docker run -e KEY=value` 传递敏感值。实测中进程参数能够显示密钥，应改用 `--env-file`、Docker Secret 或 stdin/文件挂载，并轮换已暴露的密钥。
5. 固定部署镜像版本并写入 commit SHA。当前 `latest` 镜像与工作树对“Task 是否可选”的实现不同，测试结果会随部署版本变化。

### P1：降低维护成本

1. 抽象 Claude Code 与 CodeBuddy 共用的 Session Init 状态机、资产选择和注册逻辑。
2. 拆分 `sqlite.ts`、Anthropic Handler 和通用 Handler，按存储、会话、审计、计费、流式协议等职责分层。
3. 为“全局用户设置覆盖指定 Base URL”增加启动前诊断，直接输出最终生效的上游地址。
4. 完善 macOS 宿主机与容器网络的校验：`host.docker.internal` 应在容器内验证，不能把宿主机 DNS 失败误报成服务失败。
5. 当 credit-report 后端缺失、Redis 限流 fail-open 时输出结构化健康状态，区分“主请求成功”和“附属能力降级”。

### P2：改善边界与生成物治理

1. 明确顶层包、asset import、generated schema/type 的所有权与生成流程。
2. 对大型生成文件引入稳定生成命令、校验哈希和禁止手改规则。
3. 补充启动拓扑图、交互模式限制和 settings precedence 文档。

## 9. 本次顺手修复与验证

启动过程中修复了两个 shell 脚本在中文右括号附近的变量边界问题：

```text
deploy/global-images/start-memory-core.sh
deploy/global-images/_lib.sh
```

修复方式是把 `$VAR）` 改成 `${VAR}）`，避免 shell 把中文标点附近的内容错误解析为变量名的一部分。脚本语法检查通过。

Rebase 涉及的聚焦测试通过：`2` 个测试文件、`9` 个测试用例全部通过。全量 typecheck 尚未通过，失败项属于当前分支已有的跨模块类型/依赖问题，应单独治理。

## 10. 后续使用

### 启动服务

```bash
cd tencentdb-agent-memory-dsh/deploy/global-images
./start-all.sh
```

### 启动带 Session Init 的 Claude Code

```bash
cd tencentdb-agent-memory-dsh
claude \
  --setting-sources project,local \
  --settings ../notes/claude-memory-proxy.settings.json \
  --model agnes-2.0-flash
```

### 只使用 Agnes，不经过 Agent Memory

在 CC-Switch 中保持 Agnes Provider 和 Claude Local Route 启用，直接运行：

```bash
claude
```

### 停止服务

```bash
cd tencentdb-agent-memory-dsh/deploy/global-images
./stop-all.sh
```

## 11. 补充案例测试

在前两次真实 Claude Code 会话基础上，又补充执行了以下案例。直接请求均从本地 `.admin-key` 读取认证信息，没有在命令输出或文档中打印密钥。

| 编号 | 案例 | 预期 | 实际结果 |
| --- | --- | --- | --- |
| C01 | 完整 Team/Agent/Task Header 自动初始化 | 跳过表单并直接绑定 | 通过；日志出现 `preset hit`、`initialized`，Agent/Task Context 均生成 |
| C02 | 已绑定会话不再携带身份 Header | 使用原绑定继续请求 | 通过；从 L2a 恢复，未重复初始化 |
| C03 | 已绑定会话跨 Proxy 重启 | 从持久层恢复绑定 | 通过；启动日志显示从磁盘加载状态，目标会话 L2a hit |
| C04 | 交互式选择“不关联资产” | bypass 且跳过注入 | 通过；日志显示 `user chose no-asset → bypass` |
| C05 | bypass 会话第二轮 | 不再弹表单 | 通过；保持 `bypassed=true` |
| C06 | bypass 会话跨 Proxy 重启 | 恢复 bypass 决定 | 通过；从 L2a 恢复并继续跳过所有注入 |
| C07 | 非法 Team Header | 不信任 Header | 通过；回退资产确认表单 |
| C08 | 非法 Agent Header | 不信任 Header | 通过；回退资产确认表单 |
| C09 | 缺少 Task Header | 按当前源码应允许无 Task 注册 | 运行镜像回退 Agent 选择表单，存在版本差异 |
| C10 | 过期 Task Header | 按当前源码应丢弃 Task 并宽召回 | 运行镜像回退资产确认表单，存在版本差异 |
| C11 | 错误 API Key | 鉴权失败，不进入 Session Init | 通过；HTTP 401 |
| C12 | `mem:session-reset` | 重新进入初始化流程 | 当前部署 `memCommand` 未启用，命令按普通输入处理；不适用 |
| C13 | 新会话召回上一会话调研结论 | 返回指定历史结论 | 未通过；Memory 工具协议错配 |

### 11.1 状态恢复证据

绑定会话在 Proxy 重启后出现：

```text
[session-db] hydrated ... initialized session(s) from disk
[session-recover] ... L2a hit status=initialized
```

bypass 会话重启后同样恢复为 `initialized (agent=-, task=-)`，随后明确记录 `bypassed → skipping all injection`。这说明绑定和 bypass 两种终态均能跨进程恢复。

### 11.2 镜像与源码语义差异

运行环境使用：

```text
agentmemory/memory-proxy:latest
image created: 2026-08-25
```

镜像内 `resolvePresetIdentity` 仍要求 Team、Agent、Task 三者齐全，未知 Task 会设置 mismatch。当前工作树已经把 Task 改为可选维度：有效 Team/Agent 足以直接注册，未知 Task 只记录告警并进入 Agent 范围宽召回。因此 C09、C10 的结果不能直接用来否定当前源码逻辑，但暴露了部署版本不可追溯的问题。

建议后续部署不要直接使用 `latest`，而是使用包含 commit SHA 的不可变镜像标签，并在 MemoryProxy 启动日志和健康接口中返回版本、构建时间与配置摘要。

### 11.3 自动化测试缺口

MemoryProxy 当前可发现并执行的单测为 2 个文件、9 个用例，本轮全部通过，但没有覆盖：

- Header 自动预选与非法 Header 回退；
- Task 缺失或过期时的降级；
- 用户主动 bypass 及其持久化；
- 已绑定会话跨重启恢复；
- Session Init 成功后的跨会话记忆召回。

上述案例适合固化为容器级 E2E 测试，并在发布镜像前同时对源码版本和待发布镜像执行。

## 12. 安全收尾

本次用户提供的 Agnes API Key 曾以明文出现在对话中，且启动脚本的参数传递方式可能使其出现在本机进程列表。建议立即在 Agnes 平台吊销并重建该 Key，然后更新 CC-Switch Provider；不要把新 Key 写入 Markdown、Git 仓库或命令历史。
