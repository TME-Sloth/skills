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
| silent       | `false` | 默认打开交互式配置页面。仅当用户明确要求静默、不打开配置页时传 `--silent`                                           |
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

### 非静默模式准备

默认会打开交互式拦截页。`sloth d2c --json` 可以打开系统浏览器，但 agent 不等待浏览器关闭，也不等待 `/submit` 同步返回。命令返回后，agent 继续轮询项目 `.sloth/<fileKey>/<nodeId>/` 下的任务文件和提交标记。

执行 `sloth d2c` 前先启动 Web 服务：

```bash
sloth server start
```

确认服务启动后，再执行 `sloth d2c ... --json`。静默模式不需要启动 Web 服务。

## 执行流程

```
Task Progress:
- [ ] Step 1: 执行 sloth d2c CLI 生成 chunks
- [ ] Step 2: 并行处理代码片段与聚合
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

CLI 可能返回两类成功结果。

非阻塞拦截页结果：

```json
{
  "ok": true,
  "action": "open_browser_and_poll_sloth",
  "fileKey": "...",
  "nodeId": "...",
  "convertedNodeId": "...",
  "interceptorUrl": "http://localhost:3100/auth-page?...&useBySkills=1",
  "readyForCodegen": false,
  "chunksReady": false,
  "plannedChunksDir": ".sloth/<fileKey>/<convertedNodeId>/chunks",
  "pollTargets": {
    "groupsDataPath": ".sloth/<fileKey>/<convertedNodeId>/groupsData.json",
    "componentsPath": ".sloth/components.json",
    "tasksDir": ".sloth/<fileKey>/<convertedNodeId>/tasks",
    "chunksDir": ".sloth/<fileKey>/<convertedNodeId>/chunks",
    "submissionPath": ".sloth/<fileKey>/<convertedNodeId>/submission.json"
  }
}
```

可直接消费 chunks 的结果：

```json
{
  "ok": true,
  "action": "consume_chunks",
  "fileKey": "...",
  "nodeId": "...",
  "convertedNodeId": "...",
  "readyForCodegen": true,
  "chunksReady": true,
  "chunksDir": ".sloth/<fileKey>/<convertedNodeId>/chunks"
}
```

- `action=open_browser_and_poll_sloth` 时，不要等待 CLI 或浏览器；进入“拦截页文件轮询”。
- `action=consume_chunks` 或 `chunksReady=true` 时，解析 `chunksDir` 与 `convertedNodeId`，并进入 Step 2。
- 如果 JSON 包含 `autoGroupingHandoff.requiresAutoGrouping=true`，按返回的 task 或 `tasksDir/subAgentTask-*.md` 派发 subagent。主 agent 只确认本地产物，然后继续等待用户在拦截页提交。
- `ok=false` 或非零退出码时跳转[错误排除](#错误排除)。

### Step 1.5：等待拦截页提交

拦截页提交前，agent 只处理本地任务文件，不开始生成代码。`groupsData.json` 只表示已有分组数据，不表示用户已经确认提交。

轮询规则：

1. 优先扫描 `tasksDir/subAgentTask-*.md`。如果 frontmatter 中 `status: pending`，按 `skill` 或 `type` 派发对应 subagent，并把 task 文件路径交给 subagent。
2. subagent 完成后，只重新读取它声明的本地产物，例如 `groupsData.json` 或 `.sloth/components.json`；不要在主上下文展开任务提示词细节。任务成功后对应 `subAgentTask-*.md` 必须被删除；如果产物有效但任务文件仍存在，主 agent 删除该任务文件，避免重复派发。任务失败时保留文件方便重试。
3. 如果只有 `groupsData.json`，不要开始生成代码；用户可能还在调整分组、提示词或组件映射。
4. 只有检测到拦截页提交标记后，才运行 `sloth d2c --local --json` 或返回结果中的 chunk 生成命令，生成/刷新 chunks。
5. 如果轮询超时，简短说明仍在等待用户提交或子任务完成，不要把它当作转码失败。

提交标记写在 `submission.json` 中。检测时只需要确认 JSON 满足：

```json
{ "status": "submitted", "intent": "initial-generation" }
```

不要点击拦截页提交按钮，不要用 DOM、坐标、快捷键或脚本替用户提交。

### Step 2：并行处理代码片段与聚合

以 Step 1 返回的 `chunksDir`，或 Step 1.5 在提交标记出现后生成的 `chunksDir` 为基础，启动多个 **sloth-d2c-agent** subagent，**并行执行**：

| 任务                 | 提示词路径                       |
| -------------------- | -------------------------------- |
| 代码片段处理（多个） | `{chunksDir}/{index}.md`         |
| 聚合处理             | `{chunksDir}/codeAggregation.md` |

全部 Subagent 完成后进入下一步。

### Step 3：生成最终代码并写入文件

主 Agent 收集第 2 步执行完毕的结果，结合读取 `{chunksDir}/finalGenerate.md` 的内容作为提示词转换代码，写入项目文件中。

如果 `{chunksDir}` 的上级目录存在 `tasks/subAgentTask-componentRegistration-*.md`，在写完代码后派发 `$sloth-d2c-components` 消费组件登记任务，把真实写入的组件登记到项目根目录 `.sloth/components.json`。不要调用 MCP `mark_components` 工具；Skills 场景通过本地文件写入完成组件登记。

## 错误排除

| 错误场景                     | 处理方式                                                                                                                                                                |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sloth: command not found`   | 优先执行 `pnpm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/`；没有 pnpm 时执行 `npm install -g sloth-d2c-mcp --registry=https://registry.npmjs.org/` |
| CLI 退出码非 0 / `ok:false`  | 读取 JSON 中的 `error`/`message` 字段并展示给用户                                                                                                                       |
| 文件不存在（chunksDir 为空） | 提示用户检查 fileKey 和 nodeId 是否正确，**停止执行**                                                                                                                   |
| 非静默模式未打开配置页       | 执行 `sloth server start` 启动 Web 服务后，不传 `--silent` 重试                                                                                                         |
| 拦截页已打开但没有提交标记 | 继续等待用户提交；如果超时，报告“仍在等待拦截页提交”，不要生成 chunks                                                                                                   |
| `groupsData.json` 存在但没有提交标记 | 不生成代码；继续等待用户确认/提交                                                                                                                                       |
| 出现 `subAgentTask-*.md` | 按 frontmatter 的 `skill` / `type` 派发 subagent；完成后确认本地产物并继续等待提交                                                                                           |
| 组件导入/组件登记等待中      | 消费 `subAgentTask-componentRegistration-*.md`；没有 task 时只按明确组件包/组件目录请求处理                                                                                             |
| 超时                         | 建议用户先执行 `sloth server restart` 再重试；或增加 shell 超时配置                                                                                                     |
| 403 错误                     | 未配置有效 Figma Token，提示用户执行 `sloth config` 并配置 `mcp.figmaApiKey`，或使用 `--figma-api-key`                                                                  |
| 404 错误                     | 设计稿未找到，提示用户核实 fileKey 和 nodeId                                                                                                                            |
| Node 版本过低                | 检查用户 Node 版本是否 ≥ 18                                                                                                                                             |
