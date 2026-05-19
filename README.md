# UADK 多 Agent 开发工作流

基于 [@the-forge-flow/tff-cc](https://github.com/the-forge-flow) 编排的 UADK（用户空间加速器开发套件）硬件压缩驱动多 agent 协作开发系统。

## 概述

本项目将 UADK hisi_zip 硬件压缩驱动开发拆分为 7 个专用 AI agent，通过 tff-cc 进行多 agent 编排，实现从需求分析到代码审查的全流程自动化协作。

### 7 个专用 Agent

| Agent | 职责 | Skill |
|-------|------|-------|
| 🔍 任务分析器 | 需求分解、依赖映射、复杂度评估 | [task-analyzer](.tff/skills/task-analyzer/SKILL.md) |
| 🗜️ UADK 开发者 | 驱动代码编写、SQE 填充、算法实现 | [uadk-developer](.tff/skills/uadk-developer/SKILL.md) |
| 👁️ 代码审查员 | SQE 字段验证、缓冲区安全审计 | [code-reviewer](.tff/skills/code-reviewer/SKILL.md) |
| 🚀 构建部署器 | 远程编译、驱动加载、硬件配置 | [build-deployer](.tff/skills/build-deployer/SKILL.md) |
| 🧪 测试调试器 | 往返测试、BD dump 分析、性能剖析 | [tester-debugger](.tff/skills/tester-debugger/SKILL.md) |
| 📝 文档编写者 | README、提交信息、PR 描述 | [technical-writer](.tff/skills/technical-writer/SKILL.md) |
| 🔬 元审查器 | Agent 行为审计、Skill 改进 | [meta-reviewer](.tff/skills/meta-reviewer/SKILL.md) |

### 工作流阶段

```
需求分析 → 代码开发 → 代码审查 → 构建部署 → 测试调试 → 文档编写 → 元审查
```

## 硬件环境

| 属性 | 值 |
|------|-----|
| 服务器 IP | 192.168.90.141 |
| BMC IP | 192.168.90.140 |
| 架构 | aarch64（鲲鹏 920，128 核） |
| 内存 | 256GB |
| 操作系统 | openEuler |
| 加速器 | hisi_zip（两颗设备，分别位于 NUMA node 0 和 node 1） |

## 快速开始

### 前置条件

- Node.js 20+
- tff-cc：`npm install -g @the-forge-flow/tff-cc`
- 鲲鹏服务器 SSH 访问：`ssh root@192.168.90.141`

### 配置

```bash
# 复制配置
cp tff-config/settings.yaml .tff/settings.yaml

# 复制 agent 和 skill
cp -r .tff/agents/ .tff/agents/
cp -r .tff/skills/ .tff/skills/
```

### 使用

```bash
# 阶段 1：任务分析
/tff:discuss

# 阶段 2：代码开发
/tff:execute <slice-id>

# 阶段 3：代码审查
/tff:verify <slice-id>

# 阶段 7：元审查
/tff:learn
```

## 驱动配置

```bash
# 标准开发加载
rmmod hisi_zip && modprobe hisi_zip uacce_mode=1

# 性能测试加载
rmmod hisi_zip && modprobe hisi_zip uacce_mode=1 perf_mode=1 pf_q_num=16

# 验证
cat /sys/module/hisi_zip/parameters/uacce_mode   # 必须为 1
cat /sys/module/hisi_zip/parameters/perf_mode    # 基准测试时应为 1
cat /sys/class/uacce/hisi_zip-0/available_instances
```

## 环境变量

| 变量 | 用途 | 默认值 |
|------|------|--------|
| WD_BD_DUMP | BD/SQE dump 级别（0=关, 1=关键字段, 2=完整 hex） | 0 |
| WD_COMP_EPOLL_EN | 启用 epoll 模式（降低同步 CPU） | 0 |
| LZ4_UADK_HW | 硬件开关 | 1 |
| LZ4_UADK_HW_CONCURRENCY | 硬件并发数 | 8 |
| LZ4_UADK_QUIET | 关闭调试输出 | 0 |

## 项目结构

```
uadk-agents/
├── .tff/
│   ├── agents/          # 7 个 agent 身份定义
│   │   ├── task-analyzer.md
│   │   ├── uadk-developer.md
│   │   ├── code-reviewer.md
│   │   ├── build-deployer.md
│   │   ├── tester-debugger.md
│   │   ├── technical-writer.md
│   │   └── meta-reviewer.md
│   └── skills/          # 7 个专用 skill 定义
│       ├── task-analyzer/SKILL.md
│       ├── uadk-developer/SKILL.md
│       ├── code-reviewer/SKILL.md
│       ├── build-deployer/SKILL.md
│       ├── tester-debugger/SKILL.md
│       ├── technical-writer/SKILL.md
│       └── meta-reviewer/SKILL.md
├── tff-config/
│   ├── settings.yaml    # tff-cc 配置
│   └── WORKFLOW.md      # 工作流文档
└── README.md
```

## 相关项目

- [UADK](https://github.com/Linaro/uadk) — 上游 UADK 仓库
- [tff-cc](https://github.com/the-forge-flow) — 多 agent 编排器
- [uadk-scheduler](https://github.com/IAMHCHCH/uadk-scheduler) — UADK 异构调度算法
