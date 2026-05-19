# UADK 多 Agent 独立用法指南

> 不依赖 tff-cc 编排器，直接用 Claude Code、OpenCode、OpenClaw 等工具驱动 7 个专用 agent。

## 核心原理

`.tff/skills/*/SKILL.md` 文件本身就是**完整的独立 system prompt**。任何支持自定义 system prompt 或 slash command 的 LLM 编码工具都可以直接使用它们。

```
.tff/
├── agents/          # agent 身份卡片（轻量，可选）
│   └── *.md         #   "我是谁 + 我负责什么 + 我不做什么"
└── skills/          # agent 完整操作手册（核心资产）
    └── */SKILL.md   #   "记忆 + 规则 + 工作流 + 参考数据"
```

**agents vs skills 的关系**：
- `agents/*.md`：像简历，2 秒了解 agent 能做什么
- `skills/*/SKILL.md`：像操作手册，包含所有执行细节

---

## 方式一：Claude Code

### 1.1 单 Agent 直调（最快）

将 SKILL.md 内容作为 `CLAUDE.md` 的一部分或通过 `/init` 加载：

```bash
# 方式 A：在 CLAUDE.md 中引入
# 编辑项目根目录的 CLAUDE.md，添加：
cat >> CLAUDE.md << 'EOF'
# UADK 开发角色

当用户要求开发 UADK 代码时，请加载并遵循以下 skill 定义：
- 代码开发: .tff/skills/uadk-developer/SKILL.md
- 代码审查: .tff/skills/code-reviewer/SKILL.md
- 构建部署: .tff/skills/build-deployer/SKILL.md
- 测试调试: .tff/skills/tester-debugger/SKILL.md
- 任务分析: .tff/skills/task-analyzer/SKILL.md
- 文档编写: .tff/skills/technical-writer/SKILL.md
- 元审查:   .tff/skills/meta-reviewer/SKILL.md

根据任务类型选择对应的 skill，严格遵守其中的规则。
EOF

# 方式 B：对话中直接引用
# 在 Claude Code 对话中输入：
# "请按照 .tff/skills/code-reviewer/SKILL.md 的定义，审查本次变更"
```

### 1.2 自定义 Slash Command（推荐）

在项目根目录创建 `.claude/settings.json`，将每个 agent 注册为 slash command：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": []
      }
    ]
  },
  "slashCommands": [
    {
      "name": "/uadk-analyze",
      "description": "启动任务分析器 agent",
      "prompt": "你现在是任务分析器。请严格按照 .tff/skills/task-analyzer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则和记忆，然后对用户的需求进行分解。"
    },
    {
      "name": "/uadk-develop",
      "description": "启动 UADK 开发者 agent",
      "prompt": "你现在是 UADK 开发者。请严格按照 .tff/skills/uadk-developer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则、SQE 参考和代码模式，然后开始实现。"
    },
    {
      "name": "/uadk-review",
      "description": "启动代码审查员 agent",
      "prompt": "你现在是 UADK 代码审查员。请严格按照 .tff/skills/code-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审查清单，然后开始审查。"
    },
    {
      "name": "/uadk-build",
      "description": "启动构建部署器 agent",
      "prompt": "你现在是构建部署器。请严格按照 .tff/skills/build-deployer/SKILL.md 中的定义工作。首先阅读该文件获取服务器规则和构建流程，注意必须向用户确认服务器信息。"
    },
    {
      "name": "/uadk-test",
      "description": "启动测试调试器 agent",
      "prompt": "你现在是测试调试器。请严格按照 .tff/skills/tester-debugger/SKILL.md 中的定义工作。首先阅读该文件获取测试矩阵、BD dump 和 GDB 调试规则。"
    },
    {
      "name": "/uadk-docs",
      "description": "启动文档编写者 agent",
      "prompt": "你现在是技术文档编写者。请严格按照 .tff/skills/technical-writer/SKILL.md 中的定义工作。首先阅读该文件获取提交信息和 PR 描述规则。"
    },
    {
      "name": "/uadk-meta",
      "description": "启动元审查器 agent",
      "prompt": "你现在是元审查器。请严格按照 .tff/skills/meta-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审计维度和检查清单。"
    }
  ]
}
```

使用方式：
```bash
/uadk-review   # 启动代码审查
/uadk-build    # 启动构建部署
/uadk-test     # 启动测试调试
```

### 1.3 Agent 链式协作

Claude Code 单会话中手动模拟 agent 流水线：

```bash
# 第一步：任务分析
/uadk-analyze
> 需求：为 hisi_comp 驱动添加 BD dump 功能
# → 输出任务树

# 第二步：代码开发（新会话或同一会话）
/uadk-develop
> 按照任务分析结果实现变更
# → 输出 diff

# 第三步：代码审查
/uadk-review
> 审查以下 diff：[粘贴上一步的 diff]
# → 输出审查意见

# 第四步：构建部署
/uadk-build
> 编译并部署到远程服务器
# → 构建通过/失败

# 第五步：测试
/uadk-test
> 运行往返测试验证正确性
# → 测试通过/失败 + 诊断报告

# 第六步：文档
/uadk-docs
> 为以上变更撰写提交信息
# → 合规的 commit message
```

---

## 方式二：OpenCode

[OpenCode](https://github.com/anthropics/opencode) 是 Anthropic 的开源 CLI 编码工具，支持自定义 instructions 和多种 LLM provider。

### 2.1 配置 Agent Instructions

在项目根目录创建 `.opencode/` 目录：

```bash
mkdir -p .opencode/agents
```

为每个 agent 创建 instruction 文件（引用 skill 定义）：

```bash
# .opencode/agents/uadk-developer.md
cat > .opencode/agents/uadk-developer.md << 'EOF'
请完整阅读并严格遵循以下文件中的定义：

角色定义：.tff/agents/uadk-developer.md
完整技能手册：.tff/skills/uadk-developer/SKILL.md

你就是该文件中描述的 UADK 开发者 agent。请遵循其中的所有规则、工作流程和代码模式。
EOF
```

### 2.2 OpenCode 会话中使用

```bash
# 以特定 agent 身份启动 OpenCode
opencode --instructions .opencode/agents/uadk-developer.md "实现 hisi_comp 的 BD dump 功能"

# 或在交互模式中切换
opencode
> /instructions .opencode/agents/code-reviewer.md
> 请审查 drv/hisi_comp.c 中的最新变更
```

### 2.3 多 Provider 支持

OpenCode 支持 Claude、GPT、Gemini 等 provider。Skill 文件是 provider 无关的（纯 Markdown），无需修改即可跨模型使用：

```bash
# 使用 Claude
opencode --model claude-sonnet-4-6 --instructions .opencode/agents/uadk-developer.md

# 使用 GPT
opencode --model gpt-5.1 --instructions .opencode/agents/uadk-developer.md
```

---

## 方式三：OpenClaw

[OpenClaw](https://github.com/openclaw/openclaw) 是多 agent 协作框架，支持同时运行多个专业 agent。

### 3.1 Agent 配置

在 `.openclaw/agents/` 下为每个 UADK agent 创建配置文件：

```yaml
# .openclaw/agents/uadk-developer.yaml
name: uadk-developer
description: UADK hisi_zip 压缩驱动开发者
model: claude-sonnet-4-6
system_prompt: |
  # 加载完整 skill
  {{ include ".tff/agents/uadk-developer.md" }}
  {{ include ".tff/skills/uadk-developer/SKILL.md" }}

tools:
  - read_file
  - write_file
  - execute_command
  - search_code

permissions:
  files: ["drv/", "include/", "lib/"]
  commands: ["make", "gcc", "ssh"]
```

```yaml
# .openclaw/agents/code-reviewer.yaml
name: code-reviewer
description: UADK 代码审查员
model: claude-opus-4-7
system_prompt: |
  {{ include ".tff/agents/code-reviewer.md" }}
  {{ include ".tff/skills/code-reviewer/SKILL.md" }}

tools:
  - read_file
  - search_code

permissions:
  files: ["drv/", "include/", "lib/"]
  commands: ["git", "grep"]
```

```yaml
# .openclaw/agents/build-deployer.yaml
name: build-deployer
description: 鲲鹏 ARM64 构建部署器
model: claude-sonnet-4-6
system_prompt: |
  {{ include ".tff/agents/build-deployer.md" }}
  {{ include ".tff/skills/build-deployer/SKILL.md" }}

tools:
  - read_file
  - execute_command
  - write_file

permissions:
  commands: ["ssh", "scp", "make", "modprobe", "rmmod", "cat", "ls"]
```

### 3.2 多 Agent 协作

```bash
# 启动所有 agent
openclaw serve

# 通过 API 调度
curl -X POST http://localhost:8080/task \
  -H "Content-Type: application/json" \
  -d '{
    "pipeline": [
      {"agent": "task-analyzer", "task": "分析需求：添加 BD dump 功能"},
      {"agent": "uadk-developer", "task": "实现上一阶段的任务"},
      {"agent": "code-reviewer", "task": "审查上一阶段的代码变更"},
      {"agent": "build-deployer", "task": "在远程服务器编译部署"},
      {"agent": "tester-debugger", "task": "运行测试验证"}
    ]
  }'
```

### 3.3 交互式使用

```bash
# 与特定 agent 会话
openclaw chat --agent uadk-developer

# 查看 agent 列表
openclaw list
```

---

## 方式四：通用方案（任何 LLM 工具）

### 4.1 作为 System Prompt 直接使用

Skill 文件可以直接作为 system prompt 粘贴到任何 LLM 对话中：

```bash
# 提取 skill 文件的 prompt 部分（去掉 frontmatter）
sed -n '/^---$/,/^---$/!p' .tff/skills/uadk-developer/SKILL.md | tail -n +2
```

### 4.2 Cursor / Windsurf / Copilot

在 IDE 的 `.cursorrules`、`.windsurfrules` 或 `.github/copilot-instructions.md` 中引用：

```markdown
# Cursor Rules (.cursorrules)
当用户要求开发 UADK 代码时：
- 你是 UADK 开发者。请完整阅读 .tff/skills/uadk-developer/SKILL.md
- 遵循其中的 SQE 填充规则、缓冲区安全规则和代码风格要求
- 所有代码必须在远程 ARM64 服务器编译
- 提交信息不含 AI 署名
```

### 4.3 Continue.dev (VS Code / JetBrains)

在 `~/.continue/config.json` 中添加自定义 slash command：

```json
{
  "slashCommands": [
    {
      "name": "uadk-dev",
      "description": "以 UADK 开发者身份工作",
      "prompt": "请完整阅读并严格遵循 .tff/skills/uadk-developer/SKILL.md 中的所有定义。你就是该文件描述的 UADK 开发者 agent。"
    }
  ]
}
```

---

## Agent 使用决策树

```
你的任务涉及什么？
│
├── 需要分解需求、规划任务？
│   └── → 任务分析器 (task-analyzer)
│
├── 需要写/改 UADK C 代码？
│   └── → UADK 开发者 (uadk-developer)
│       ├── 涉及 SQE 结构体字段？ → 参考 SQE 位位置速查表
│       ├── 涉及新算法？ → 参考 SQE 填充模式
│       └── 涉及缓冲区？ → 参考缓冲区安全规则
│
├── 需要审查代码变更？
│   └── → 代码审查员 (code-reviewer)
│       ├── SQE 字段变更？ → SQE 审查清单
│       └── 缓冲区变更？ → 缓冲区安全清单
│
├── 需要在 ARM64 服务器编译/部署？
│   └── → 构建部署器 (build-deployer)
│       └── 注意：必须先向用户确认服务器信息
│
├── 测试失败 / segfault / 性能问题？
│   └── → 测试调试器 (tester-debugger)
│       ├── 硬件错误？ → BD dump (WD_BD_DUMP=1/2)
│       ├── 崩溃？ → GDB + core dump
│       └── 性能？ → perf 剖析
│
├── 需要写提交信息 / PR 描述？
│   └── → 技术文档编写者 (technical-writer)
│       └── 注意：提交信息不含 AI 署名
│
└── 想审计 agent 工作质量？
    └── → 元审查器 (meta-reviewer)
```

---

## 快速参考：各 Agent 一行定位

```bash
# 需要时直接用以下 prompt 调用
"请以 任务分析器 身份工作：阅读 .tff/skills/task-analyzer/SKILL.md"
"请以 UADK开发者 身份工作：阅读 .tff/skills/uadk-developer/SKILL.md"
"请以 代码审查员 身份工作：阅读 .tff/skills/code-reviewer/SKILL.md"
"请以 构建部署器 身份工作：阅读 .tff/skills/build-deployer/SKILL.md，先确认服务器信息"
"请以 测试调试器 身份工作：阅读 .tff/skills/tester-debugger/SKILL.md"
"请以 文档编写者 身份工作：阅读 .tff/skills/technical-writer/SKILL.md"
"请以 元审查器 身份工作：阅读 .tff/skills/meta-reviewer/SKILL.md"
```
