# TME Sloth Skills

## 安装

在项目或本机环境中，执行：

```bash
npx skills add git@github.com:TME-Sloth/skills.git
```

按 `skills` CLI 的提示完成安装；通常会将本仓库中的 skill 同步到你配置的 skills 目录。

### 环境要求

- 已安装 **Node.js**（建议 ≥ 18），以便使用 `npx`
- 已安装并配置好对应的 **skills** 工具链（由 `npx skills` 提供）

## 包含的 Skills

| 目录                                            | 说明                                                                      |
| ----------------------------------------------- | ------------------------------------------------------------------------- |
| [sloth-d2c-skills](./sloth-d2c-skills/)         | Figma 设计稿转前端代码（D2C），配合 `sloth d2c` CLI                       |
| [sloth-d2c-auto-grouping](./sloth-d2c-auto-grouping/) | 消费 `autoGrouping.md` 并写入 `.sloth/**/groupsData.json` 自动分组结果 |
| [sloth-d2c-components](./sloth-d2c-components/) | 消费 `marked-components.todo.json` 并维护 `.sloth/components.json` 组件库 |

## Agents

| 目录                                                     | 说明                                          |
| -------------------------------------------------------- | --------------------------------------------- |
| [agents/sloth-d2c-agent.md](./agents/sloth-d2c-agent.md) | D2C 代码片段转换子 Agent，由主 Skill 并行调度 |

## 仓库地址

- SSH：`git@github.com:TME-Sloth/skills.git`
- HTTPS：`https://github.com/TME-Sloth/skills.git`

许可证与贡献方式以团队规范为准。
