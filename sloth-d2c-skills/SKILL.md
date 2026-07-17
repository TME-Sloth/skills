---
name: sloth-d2c-skills
description: 将Figma设计稿转换为前端组件代码（Design to Code）。通过 `sloth d2c` CLI 获取设计稿数据，分片处理并生成最终代码。当用户提到Figma转代码、设计稿转代码、D2C、design to code、生成页面时使用。
allowed-tools: Bash, Task, Read, Write, Edit, Glob, Grep
disable: false
---

# Figma 设计稿转代码（D2C）

## 前置校验

### 必需参数

| 参数    | 说明           |
| ------- | -------------- |
| fileKey | Figma 文件 Key |
| nodeId  | Figma 节点 ID  |

缺少以上参数时，提示用户提供。

### 可选参数

| 参数         | 默认值  | 使用时机                                                                                                            |
| ------------ | ------- | ------------------------------------------------------------------------------------------------------------------- |
| framework    | 自动    | 用户明确指定目标框架时传入，取值：`react` / `vue` / `ios-oc` / `ios-swift` / `kuikly` / `taro` / `uniapp` / `hippy` |
| depth        | 自动    | 仅当用户显式要求限制节点树遍历深度时传入，否则不加                                                                  |
| local        | `false` | **默认不要加**。仅当用户明确要求"使用本地缓存"才传入                                                                |
| update       | `false` | 仅当用户明确表示"修改/更新之前生成的代码"时传入；新建代码时一律不传                                                 |
| silent       | `false` | 默认使用交互模式并打开配置页。仅当用户明确要求静默、不打开配置页时传 `--silent`                                     |
| autoGrouping | `false` | 仅当用户明确要求"自动分组"、"AI 分组"、"automatic grouping"、"automatic splitting" 时传 `--auto-grouping`            |

> ⚠️ `local` 与 `update` 都是**显式触发**参数，默认一律不传。不要因为"为了更快"而主动加 `--local`——运行没有缓存会直接失败。

### 环境检查

执行 `sloth --version` 确认 CLI 可用。

如果 `sloth` 不存在，先自动安装：

```bash
pnpm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/
```

如果当前环境没有 `pnpm`，使用 npm：

```bash
npm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/
```

安装后再次执行 `sloth --version` 校验，仍不可用则跳转[错误排除](#错误排除)。

### 交互模式准备

默认使用交互模式：打开拦截页供用户确认配置、分组和组件标记。`sloth d2c --json` 返回后，agent 打开拦截页并运行返回的阻塞式 wait 命令。

执行 `sloth d2c` 前先启动 Web 服务：

```bash
sloth server start
```

确认服务启动后，再执行 `sloth d2c ... --json`。静默模式不需要启动 Web 服务。

## 执行流程

```
Task Progress:
- [ ] Step 1: 执行 sloth d2c CLI 生成 chunks
- [ ] Step 2: 消费 chunks/prompts
- [ ] Step 3: 生成最终代码并写入文件
```

### Step 1：执行 `sloth d2c` CLI

在工作区根目录运行（Bash）：

```bash
sloth d2c \
  --file-key <fileKey> \
  --node-id <nodeId> \
  [--framework <react|vue|ios-oc|ios-swift|kuikly|taro|uniapp|hippy>] \
  [--depth <n>] \
  [--local] \
  [--update] \
  [--auto-grouping] \
  [--silent] \
  --json
```

CLI 只有两种模式：交互模式会打开拦截页，静默模式不唤起拦截页。CLI 或 `wait.command` 成功后常见三类 action：

交互模式结果（有拦截页）：

```json
{
  "ok": true,
  "action": "open_browser_and_wait",
  "interceptorUrl": "http://localhost:3100/auth-page?...&useBySkills=1",
  "wait": {
    "command": "sloth d2c wait --file-key '...' --node-id '...' --run-id '...' --json"
  }
}
```

chunks 已就绪结果（可直接消费 chunks）：

```json
{
  "ok": true,
  "action": "consume_chunks",
  "chunksDir": ".sloth/<fileKey>/<nodeId>/chunks"
}
```

可选 AI 自动分组任务（用户通过 `--auto-grouping` 或拦截页显式启用时可能返回）：

```json
{
  "ok": true,
  "action": "handle_subagent_task",
  "task": {
    "path": ".sloth/<fileKey>/<nodeId>/tasks/subAgentTask-autoGrouping-<id>.md",
    "skill": "sloth-d2c-auto-grouping"
  }
}
```

- `action=open_browser_and_wait` 时，先打开 `interceptorUrl`，再运行 `wait.command` 阻塞等待事件。
- `action=handle_subagent_task` 时，先按 `task.skill` 派发 `task.path`，不能进入 Step 2。
- `action=consume_chunks` 时，读取 `chunksDir` 并进入 Step 2。静默模式会直接到这里；交互模式在用户提交完成后也会到这里。
- wait 会先返回已经存在的 pending task，因此任务和提交都统一通过 `wait.command` 接收。
- `ok=false` 或非零退出码时跳转[错误排除](#错误排除)。

### Step 1.5：等待拦截页提交

拦截页提交前，agent 只处理 wait 返回的任务，不开始生成代码。`groupsData.json` 只表示已有分组数据，不表示用户已经确认提交。

等待规则：

1. 打开拦截页后，在工作区根目录运行返回的 `wait.command`。命令阻塞等待，不设置业务超时，也不要自行扫描 task 或 `submission.json`。
2. 返回 `action: "handle_subagent_task"` 时，把 `task.path` 交给 `task.skill` 对应的 subagent。只重新读取 subagent 声明的本地产物，不在主上下文展开任务正文。
3. 任务成功并校验产物后，确认对应 task 文件已删除，再次运行同一个 `wait.command`；任务失败时保留 task 并停止，避免立即重复接收。
4. 返回 `action: "consume_chunks"` 时，从结果中的 `chunksDir` 进入 Step 2。
5. 返回 `action: "error"` 或 `ok: false` 时，报告 `error` 并停止。
6. 首次配置阶段页面不判断 listener，也不会因为 wait 连接暂时结束而自动复制 Prompt；每个 task 完成后应立即重新运行同一个 `wait.command`。
7. Agent 主动不再等待时，终止 wait 命令即可。以后需要恢复首次流程时，重新运行同一个 wait 命令；它会立即返回已经落盘的 task 或 submitted 事件。

可选 AI 自动分组只在本 skill 中做入口路由：

1. 当 `action: "handle_subagent_task"` 且 `task.skill: "sloth-d2c-auto-grouping"` 时，启动聚焦 subagent，要求它加载 `$sloth-d2c-auto-grouping` 并仅传入 `task.path`。
2. 主 Agent 不执行分组算法，也不展开 task 正文或完整分组 JSON；只重新读取 subagent 声明的 `groupsData.json` 确认结果。
3. 交互模式下，任务成功且 task 文件已删除后，重新运行同一个 `wait.command`；静默模式直接返回任务时，执行返回的 `resumeCommand` 恢复 chunks 生成。
4. 分组失败时保留 task 文件并停止，不绕过分组直接生成代码。

不要点击拦截页提交按钮，不要用 DOM、坐标、快捷键或脚本替用户提交。

### Step 2：消费 chunks/prompts

从 Step 1 返回的 `chunksDir`，或 Step 1.5 提交完成后的 `chunksDir` 读取 prompts。先处理 `0.md`、`1.md` 这类已存在的 group chunks，再处理 `codeAggregation.md` 和 `finalGenerate.md`；静默模式可能没有数字 chunk。

处理规则：

1. 如果存在 `{chunksDir}/{index}.md`，可把这些 group chunk 派发给 **sloth-d2c-agent** subagent 并行处理。
2. 如果不存在数字 chunk，不要等待或报错，直接处理 `{chunksDir}/codeAggregation.md`。
3. 最后读取 `{chunksDir}/finalGenerate.md`，进入 Step 3。

### Step 3：生成最终代码并写入文件

主 Agent 收集第 2 步执行完毕的结果，结合 `{chunksDir}/finalGenerate.md` 的内容作为提示词转换代码，写入项目文件中。

如果 `{chunksDir}` 的上级目录存在 `tasks/subAgentTask-componentRegistration-*.md`，在写完代码后派发 `$sloth-d2c-components` 消费组件登记任务，把真实写入的组件登记到项目根目录 `.sloth/components.json`。组件登记通过本地文件完成，不调用 MCP `mark_components` 工具。

## 错误排除

| 错误场景                     | 处理方式                                                                                                                                                                |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sloth: command not found`   | 优先执行 `pnpm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/`；没有 pnpm 时执行 `npm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/` |
| CLI 退出码非 0 / `ok:false`  | 读取 JSON 中的 `error` 字段并展示给用户                                                                                                                               |
| 文件不存在（chunksDir 为空） | 提示用户检查 fileKey 和 nodeId 是否正确，**停止执行**                                                                                                                   |
| 交互模式未打开配置页       | 执行 `sloth server start` 启动 Web 服务后，不传 `--silent` 重试                                                                                                         |
| 拦截页已打开但尚未提交 | 运行 `wait.command` 持续阻塞等待，不自行点击提交或开始生成                                                                                       |
| `groupsData.json` 存在但没有 submitted 事件 | 不生成代码；继续运行 wait 命令等待用户确认/提交                                                                                                                               |
| wait 返回 `handle_subagent_task` | 按 `task.skill` 派发 `task.path`；完成后确认本地产物和 task 删除，再次运行 wait                                                                                       |
| 组件导入/组件登记等待中      | 消费 `subAgentTask-componentRegistration-*.md`；没有 task 时只按明确组件包/组件目录请求处理                                                                                             |
| wait 连接失败                 | 确认 `sloth server start` 已运行；不要降级为手工文件轮询                                                                                                                       |
| 403 错误                     | 未配置有效 Figma Token，提示用户执行 `sloth config` 并配置 `mcp.figmaApiKey`，或使用 `--figma-api-key`                                                                  |
| 404 错误                     | 设计稿未找到，提示用户核实 fileKey 和 nodeId                                                                                                                            |
| Node 版本过低                | 检查用户 Node 版本是否 ≥ 18                                                                                                                                             |
