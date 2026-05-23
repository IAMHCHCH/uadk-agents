# UADK 多 Agent 开发工作流 — 完整示例

> 以"为 hisi_zip 驱动新增 lz4 压缩算法硬件卸载支持"为例，演示两种使用路径的完整流程。

## 路径选择

| 路径 | 编排方式 | 适合场景 |
|------|---------|---------|
| **A. tff-cc 编排** | `/tff-cc:new → discuss → plan → execute → verify → ship → learn` | 大型功能，需要严格 agent 流水线和质量门禁 |
| **B. 直调 Agent** | 手动按顺序加载各 agent 的 SKILL.md | 中小任务，快速迭代，零配置 |

---

## 路径 A：tff-cc 全自动编排

### 原理

tff-cc 是 Claude Code 插件，通过 `claude plugin install` 安装后以 slash command 形式工作。它按阶段调度内置 agent，状态通过 SQLite 持久化，上游输出自动作为下游的上下文。

```
你的需求
   │
   ▼
┌─────────────┐
│  /tff-cc:new  │  项目初始化（仅首次）
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│/tff-cc:discuss│ ──► │ /tff-cc:plan  │ ──► │/tff-cc:execute│
│  需求讨论   │     │  任务规划   │     │  代码开发   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                                │
                    ┌─────────────┐              │
                    │ /tff-cc:ship  │ ◄──┐        │
                    │  PR 提交    │    │        ▼
                    └──────┬──────┘   ┌─────────────┐
                           │     ◄── │/tff-cc:verify │
                           │          │  代码审查   │
                           ▼          └─────────────┘
                    ┌─────────────┐
                    │ /tff-cc:learn │
                    │  元审查     │
                    └─────────────┘
```

### 阶段 0：环境准备

```bash
# 1. 添加 tff-cc 的 marketplace（首次使用时执行）
claude plugin marketplace add MonsieurBarti/tff-mono

# 2. 安装 tff-cc 插件
claude plugin install tff-cc@the-forge-flow

# 3. 重启 Claude Code 使插件生效
#    退出当前 Claude Code 会话，重新启动即可

# 4. 在 Claude Code 交互会话中验证插件
/tff-cc:help
# 输出包含：
#   /tff-cc:new        — 初始化项目
#   /tff-cc:discuss    — 需求讨论
#   /tff-cc:plan       — 任务规划
#   /tff-cc:execute    — 执行开发
#   /tff-cc:verify     — 代码审查
#   /tff-cc:ship       — PR 提交
#   /tff-cc:learn      — 元审查与学习
```

> **⚠️ 注意**：不要使用 `npm install -g @the-forge-flow/tff-cc`。tff-cc 是 Claude Code 插件，应通过 `claude plugin install` 安装。插件安装后 `tff-tools` CLI 可通过 `$PLUGIN_ROOT/bin/tff-tools` 访问（在 Claude Code 启动的 shell 中自动加入 PATH）。

#### Node.js 版本与 native binding 兼容性

tff-cc 插件预编译了 `better-sqlite3` 的 native binding，对应 **Node 22**（ABI 127）。如果你的 Node 版本较新（如 25.x，ABI 141），运行 `tff-tools` 时会报 `NATIVE_BINDING_FAILED` 错误：

```
The module 'better_sqlite3.darwin-x64.node' was compiled against a different Node.js version
using NODE_MODULE_VERSION 127. This version of Node.js requires NODE_MODULE_VERSION 141.
```

**解决方法**：用当前 Node 版本重新编译 native binding（需要 Python 3.8+）：

```bash
# 1. 确保有 Python 3.8+（node-gyp 需要）
python3 --version  # 如果 < 3.8，安装: brew install python@3.12

# 2. 在临时目录重新编译 better-sqlite3
cd /tmp && npm init -y --silent
PYTHON=$(which python3.9 || which python3.11 || which python3.12) \
  npm install better-sqlite3@12.8.0

# 3. 替换预编译的 binding
PLUGIN_ROOT="$HOME/.claude/plugins/cache/the-forge-flow/tff-cc/1.1.3"
cp /tmp/node_modules/better-sqlite3/build/Release/better_sqlite3.node \
  "$PLUGIN_ROOT/dist/cli/better_sqlite3.$(node -e "console.log(process.platform+'-'+process.arch)").node"

# 4. 验证
$PLUGIN_ROOT/bin/tff-tools --version
```

#### 备用：切换 Node 版本

如果你有 nvm/fnm，更简单的方式是切换到 Node 22：

```bash
nvm install 22 && nvm use 22   # 或: fnm install 22 && fnm use 22
```

### 阶段 1：`/tff-cc:new` — 项目初始化

首次在项目中使用 tff-cc 时，必须先初始化。

#### 初始化流程

```
/tff-cc:new
```

tff-cc 会引导你：
1. 检测代码库（有源码 → 可选分析架构；无源码 → 跳过）
2. 确认项目名称和愿景
3. 执行 `tff-tools project:init` 创建 `.tff/` 目录

以本项目的实际初始化为例（在 [uadk](https://github.com/Linaro/uadk) C 代码库中开发调度新功能）：

```
项目名称：uadk_sched
项目愿景：新增 loop、hungry、instr 三种调度模式功能，大幅提升 UADK 硬件加速调度性能
```

#### 关键限制：不能在默认分支上初始化

`tff-tools` 有**默认分支守卫**——所有写操作拒绝在 `master`/`main` 分支上执行：

```
{"ok":false,"error":{"code":"REFUSED_ON_DEFAULT_BRANCH",
  "message":"Refusing to run \"project:init\" on default branch \"master\"."}}
```

**解决**：先创建工作分支再初始化：

```bash
git checkout -b milestone/init-project
# 然后重新执行 /tff-cc:new
```

#### 初始化后的项目结构

```bash
uadk/
├── .tff/                    # → 实际上是指向 ~/.tff/{projectId}/ 的符号链接
│   ├── PROJECT.md           # 项目愿景（markdown 权威来源）
│   ├── settings.yaml        # 模型配置、质量门禁、agent 工作流定义
│   ├── state.db             # SQLite 状态管理（STATE.md 由此派生）
│   ├── agents/              # 自定义 agent 身份定义（7 个 .md）
│   ├── skills/              # 自定义 skill 定义（7 个目录，每个含 SKILL.md）
│   ├── milestones/          # 里程碑（M01, M02, ...）
│   ├── worktrees/           # git worktree（gitignored）
│   └── journal/             # 操作日志
```

> **注意**：
> - 如果项目已有 `.tff/` 目录（已初始化过），可跳过此步骤，直接进入 `/tff-cc:discuss`
> - `.tff/` 是符号链接，实际数据存储在 `~/.tff/{projectId}/`，不会污染 git 仓库
> - Agent 和 Skill 定义可以从参考项目（如 `uadk-agents/`）复制到 `.tff/` 目录中

### 阶段 2：`/tff-cc:discuss` — 需求分解

在 Claude Code 中输入：

```
/tff-cc:discuss

需求：为 hisi_zip 驱动新增 lz4 压缩算法硬件卸载支持。
约束：
- lz4: MINMATCH=4, LASTLITERALS=12, max_offset=65535
- 输出缓冲区 ≥ 2× 输入（min 8192 bytes）
- 参考 lz4_uadk 应用层的实现思路
- 不能破坏现有硬件结构体 ABI
```

**tff-cc 做的事情**：启动内置 agent，进行需求讨论和范围界定，将需求分解为可执行的 slice。

**输出**：

```
╔══════════════════════════════════════════════════════╗
║           UADK lz4 支持 — 任务分解                    ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  Slice #1 [SS]  算法枚举注册                         ║
║  Agent:     uadk-developer                           ║
║  文件:      include/wd_comp.h                        ║
║  内容:      在 wd_comp_alg_type 中添加 WD_LZ4        ║
║  依赖:      无                                        ║
║  验证:      编译通过，枚举值不冲突                    ║
║                                                      ║
║  Slice #2 [SSS] SQE 填充函数实现                      ║
║  Agent:     uadk-developer                           ║
║  文件:      drv/hisi_comp.c                          ║
║  内容:      实现 fill_buf_lz4()，ops 表注册           ║
║  依赖:      Slice #1                                  ║
║  验证:      WD_BD_DUMP=2 检查 SQE 字段正确            ║
║                                                      ║
║  Slice #3 [S]   代码审查                              ║
║  Agent:     code-reviewer                            ║
║  文件:      include/wd_comp.h, drv/hisi_comp.c       ║
║  依赖:      Slice #1, #2                              ║
║  验证:      SQE 审查清单全绿                          ║
║                                                      ║
║  Slice #4 [S]   远程构建                              ║
║  Agent:     build-deployer                           ║
║  内容:      编译 + modprobe hisi_zip uacce_mode=1    ║
║  依赖:      Slice #3 (审查通过后)                     ║
║  验证:      零警告编译，available_instances > 0       ║
║                                                      ║
║  Slice #5 [SS]  往返测试                              ║
║  Agent:     tester-debugger                          ║
║  内容:      lz4 压缩→解压，0~8M 边界测试              ║
║  依赖:      Slice #4                                  ║
║  验证:      100% 数据还原，BD dump 无异常              ║
║                                                      ║
║  Slice #6 [S]   提交信息                              ║
║  Agent:     technical-writer                         ║
║  依赖:      Slice #3, #5 均通过                       ║
║  验证:      合规 commit message                       ║
║                                                      ║
╚══════════════════════════════════════════════════════╝

📍 依赖图：
   #1 ──► #2 ──► #3 ──► #4 ──► #5 ──► #6

📊 总预估: 6 slices | 复杂度: SS/SSS | 预计 4-6 轮交互
```

### 阶段 3：`/tff-cc:plan` — 任务规划

在 discuss 确认需求后，使用 plan 进行详细的任务分解和规划：

```
/tff-cc:plan
```

tff-cc 会将讨论结果细化为带依赖关系的执行计划，定义每个 slice 的入口/出口条件和验收标准。

### 阶段 4：`/tff-cc:execute` — 逐 Slice 执行

tff-cc 按依赖拓扑顺序调度 agent 执行代码开发。每个 slice 完成后自动标记状态，下游 slice 就绪后才能执行。

```
# 在 Claude Code 交互会话中执行
/tff-cc:execute
```

tff-cc 启动内置 executor agent，从 SQLite 读取上下文，执行代码变更：

```c
// include/wd_comp.h
enum wd_comp_alg_type {
    WD_DEFLATE,
    WD_ZLIB,
    WD_GZIP,
    WD_LZ77_ZSTD,
    WD_LZ4,        // ← 新增
    WD_LZ77_ONLY,
    WD_COMP_ALG_MAX,
};
```

```
# 当前 slice 完成后，继续执行下一个 slice
/tff-cc:execute
```

```c
// drv/hisi_comp.c — ops 表注册
[WD_LZ4] = {
    .alg_name       = "lz4",
    .fill_buf       = fill_buf_lz4,     // ← 新函数
    .parse_result   = parse_lz77_result, // 复用 lz77 解析逻辑
    .min_src_size   = 0,
    .min_dst_size   = 8192,
    .sqe_type       = SQE_TYPE_V1,
    .hash_type      = HZ_LZ4,
},

// fill_buf_lz4 实现
static int fill_buf_lz4(handle_t h_qp, struct hisi_zip_sqe *sqe,
                         struct wd_comp_msg *msg)
{
    struct hisi_comp_sqe_addr addr = {0};
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;

    // lz4 特有约束：output ≥ 2× input
    if (msg->avail_out < msg->in_cons * 2)
        return -WD_EINVAL;

    sqe->input_data_length = msg->in_cons;
    sqe->dest_avail_out = msg->avail_out;

    addr.src_addr = msg->src;
    addr.dst_addr = msg->dst;
    zip_mem_map(mm_ops, sqe, &addr);

    sqe->dw9 = (HZ_LZ4 & HZ_REQ_TYPE_MASK) |
               (msg->buf_type << BUF_TYPE_SHIFT);
    return 0;
}
```

> **注意**：tff-cc 不自动写入文件 — 它输出 diff，你在编辑器中确认后生效。人对代码有最终控制权。

### 阶段 5：`/tff-cc:verify` — 代码审查

```
/tff-cc:verify
```

tff-cc 启动内置 verifier agent，对已完成的 slice 进行验收标准验证：

```
╔══════════════════════════════════════════════════════╗
║              Slice #3 审查报告                        ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  🟡 应修复：fill_buf_lz4 缺少 avail_out 下限检查     ║
║  文件: drv/hisi_comp.c:1245                          ║
║  原因: lz4 要求 output >= 2× input，当前未验证        ║
║  修复: if (msg->avail_out < msg->in_cons * 2)        ║
║          return -WD_EINVAL;                           ║
║                                                      ║
║  🔴 阻塞：dw9 中 buffer_type 位偏移未定义常量          ║
║  当前: sqe->dw9 = ... | (2 << 8);                    ║
║  修复: 使用 #define BUF_TYPE_LZ4 2                    ║
║                                                      ║
║  💭 建议：fill_buf_lz4 的 last_literals 逻辑           ║
║  可加一行注释解释 12 字节边界的硬件原因                ║
║                                                      ║
║  审批: 🔴 需修改 (1 blocking, 1 should-fix)          ║
║                                                      ║
╚══════════════════════════════════════════════════════╝
```

审查不通过 → 回到 `uadk-developer` 修复 → 重新审查。这是 tff-cc 的内置反馈循环，有阻塞项时无法执行后续 slice。

### 阶段 6：`/tff-cc:execute` — 继续执行（构建 + 测试）

审查通过后继续执行后续 slice：

```
/tff-cc:execute   # 继续执行构建部署 slice
```

此时代理**主动询问**服务器信息：

```
🔐 构建部署器：需要远程服务器信息才能继续。

请在对话中提供：
  - 服务器 IP：
  - SSH 用户名：
  - 认证方式：(密钥路径 / 密码 / 免密)
  - UADK 源码路径：

建议先配置免密登录：
  ssh-copy-id {user}@{server}
```

你回复后 agent 执行：

```bash
# SSH 到服务器编译
ssh user@server "cd /path/to/uadk && \
  git pull && make clean && \
  ./autogen.sh && ./configure && make -j$(nproc) 2>&1"
# [输出] 0 errors, 0 warnings ✅

# 加载驱动
ssh user@server "rmmod hisi_zip 2>/dev/null; \
  modprobe hisi_zip uacce_mode=1 perf_mode=1"

# 验证设备
ssh user@server "cat /sys/class/uacce/hisi_zip-0/available_instances"
# [输出] 1024 ✅
```

```
/tff-cc:execute   # 继续执行测试 slice
```

测试 agent 自动运行并解析结果：

```
[PASS] input=0B      compressed=0B     decomp=0B     match ✅
[PASS] input=1B      compressed=1B     decomp=1B     match ✅
[PASS] input=256B    compressed=128B   decomp=256B   match ✅
[PASS] input=1KB     compressed=512B   decomp=1KB    match ✅
[PASS] input=64KB    compressed=32KB   decomp=64KB   match ✅
[PASS] input=1MB     compressed=512KB  decomp=1MB    match ✅
[PASS] input=8MB     compressed=4MB    decomp=8MB    match ✅
7/7 passed
```

若测试失败，agent 自动切换调试模式：

```bash
# 自动启用 BD dump
WD_BD_DUMP=2 ./lz4_test 2>&1 | tee bd_full.log
# 分析 SQE 字段：status 错误码？consumed vs input_length？
# segfault → ulimit -c unlimited → gdb ./lz4_test /tmp/core.*
```

### 阶段 7：`/tff-cc:ship` — PR 提交

```
/tff-cc:ship
```

tff-cc 执行代码审查、安全审计，并生成 PR。

```
feat: add lz4 hardware offload support to hisi_zip driver

Register lz4 algorithm in hisi_comp ops table with fill_buf_lz4
SQE filling function. LZ4 constraints enforced: MINMATCH=4,
LASTLITERALS=12, max_offset=65535, output buffer >= 2x input.

Signed-off-by: Developer <dev@example.com>
```

### 阶段 8：`/tff-cc:learn` — 元审查

```
/tff-cc:learn
```

`meta-reviewer` 审计全流程并给出改进建议：

```
╔══════════════════════════════════════════════════════╗
║           会话审计报告 #lz4-2026-0520                  ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  Agent 评分:                                         ║
║  task-analyzer    8/10 — 分解粒度好，漏了 SGL 场景   ║
║  uadk-developer   7/10 — 代码正确，首次漏了边界检查  ║
║  code-reviewer    9/10 — 准确捕获缓冲区验证缺失       ║
║  build-deployer   8/10 — 编译部署顺利                 ║
║  tester-debugger  9/10 — 测试覆盖完整，边界 0B 也测了 ║
║                                                      ║
║  Skill 改进建议:                                      ║
║  1. uadk-developer: 记忆库应补充 lz4 的 SQE 填充模板 ║
║  2. tester-debugger: 测试矩阵应增加 lz4 特定边界      ║
║     (LASTLITERALS=12, MINMATCH=4)                    ║
║  3. docs/uadk-reference.md: 建议新增 lz4 算法专项章节 ║
║                                                      ║
╚══════════════════════════════════════════════════════╝
```

### 实际初始化案例：uadk_sched

以下是本项目（在 [UADK](https://github.com/Linaro/uadk) C 代码库中开发异构调度功能）的完整初始化过程，可作为参考。

**项目配置：**

| 项目属性 | 值 |
|---------|-----|
| 代码库路径 | `uadk/`（152 个 C 源文件，autotools 构建） |
| tff 项目名 | `uadk_sched` |
| 项目愿景 | 新增 loop、hungry、instr 三种调度模式，大幅提升调度性能 |
| 核心目标文件 | `wd_sched.c`（2095 行） |

**Agent/Workflow 配置：**

从参考配置项目 `uadk-agents/.tff-cc/` 复制 agent 和 skill 定义到 tff 项目目录：

```bash
cp uadk-agents/.tff-cc/agents/*.md .tff/agents/
cp -r uadk-agents/.tff-cc/skills/* .tff/skills/
```

最终 `.tff/settings.yaml` 包含 7 个自定义 agent 和 7 阶段工作流：

```
分析(task-analyzer) → 开发(uadk-developer) → 审查(code-reviewer)
  → 构建(build-deployer) → 测试(tester-debugger)
    → 文档(technical-writer) → 元审查(meta-reviewer)
```

**初始化时遇到的典型问题：**

| 问题 | 原因 | 解决 |
|------|------|------|
| `REFUSED_ON_DEFAULT_BRANCH` | tff-tools 拒绝在 master 上执行写操作 | `git checkout -b milestone/init-project` |
| `NATIVE_BINDING_FAILED` | Node 25.x 与预编译 binding (Node 22 ABI) 不匹配 | Python 3.9+ 重新编译 better-sqlite3（见阶段 0） |

---

## 路径 B：直调 Agent（手动编排）

不依赖 tff-cc，在 Claude Code 中手动依次加载 agent 完成全流程。

### 第 1 步：任务分析

```
请以任务分析器身份工作。首先阅读 .tff/skills/task-analyzer/SKILL.md 
获取完整规则。同时参考 docs/uadk-reference.md 第1节(架构)、第2节(项目结构)、
第4节(API)。我需要在 hisi_zip 驱动中新增 lz4 压缩算法的硬件卸载支持。
lz4 算法约束：MINMATCH=4, LASTLITERALS=12, 最大offset=65535, 
输出缓冲区至少为输入的 2 倍。请分解为原子任务。
```

### 第 2 步：代码开发

```
请以 UADK 开发者身份工作。阅读 .tff/skills/uadk-developer/SKILL.md 
获取完整规则和 SQE 参考。同时参考 docs/uadk-reference.md 第5节(压缩驱动)、
第6节(SQE描述符)。

需要做：
1. 在 include/wd_comp.h 的 enum wd_comp_alg_type 中添加 WD_LZ4
2. 在 drv/hisi_comp.c 的 ops 表中注册 lz4 算法
3. 实现 fill_buf_lz4() — 参考 fill_buf_lz77() 的模式，
   适配 lz4 的 MINMATCH=4, LASTLITERALS=12, max_offset=65535,
   SQE type 使用 V1(0x0), dw9 中 buffer_type 设置正确

注意：output buffer >= 2x input (min 8192 bytes)
不可修改硬件结构体布局
```

### 第 3 步：代码审查

```
请以代码审查员身份工作。阅读 .tff/skills/code-reviewer/SKILL.md，
同时参考 docs/uadk-reference.md 第6节(SQE字段定义)和第9节(内存管理)。
审查刚刚的变更，逐项验证 SQE 审查清单，给出 🔴/🟡/💭 分级意见。
```

### 第 4 步：构建部署

```
请以构建部署器身份工作。阅读 .tff/skills/build-deployer/SKILL.md。
代码在 drv/hisi_comp.c 和 include/wd_comp.h。需要：
- 在 {你的服务器}:{你的路径} 编译
- 加载驱动 modprobe hisi_zip uacce_mode=1 perf_mode=1
- 验证 available_instances > 0

请先告诉我服务器的连接信息我来提供。
```

### 第 5 步：测试验证

```
请以测试调试器身份工作。阅读 .tff/skills/tester-debugger/SKILL.md，
同时参考 docs/uadk-reference.md 第15节(调试诊断)和第6节(错误码表)。
对 lz4 算法运行往返测试，覆盖输入大小 0,1,256,1K,64K,1M,8M。
如有失败，先启用 WD_BD_DUMP=1 检查 SQE 字段。
segfault 则用 GDB + core dump 定位。
```

### 第 6 步：文档合入

```
请以文档编写者身份工作。阅读 .tff/skills/technical-writer/SKILL.md。
为以上变更生成合规的 commit message（不含 AI 署名）。
```

### 第 7 步：元审查（可选）

```
请以元审查器身份工作。阅读 .tff/skills/meta-reviewer/SKILL.md。
对本次 lz4 开发的 agent 协作质量进行审计。
```

---

## 配置 Slash Command（可选，路径 B 提速）

在项目的 `.claude/commands/` 目录下创建 Markdown 文件，每个文件即为一个自定义 slash command，文件名即命令名：

```bash
# 创建命令目录
mkdir -p .claude/commands

# 创建各 agent 的快捷命令
cat > .claude/commands/uadk-analyze.md << 'EOF'
你现在是任务分析器。请严格按照 .tff/skills/task-analyzer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则和记忆，同时参考 docs/uadk-reference.md。
EOF

cat > .claude/commands/uadk-develop.md << 'EOF'
你现在是 UADK 开发者。请严格按照 .tff/skills/uadk-developer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则、SQE 参考和代码模式，参考 docs/uadk-reference.md 第5-7节和第9-10节。
EOF

cat > .claude/commands/uadk-review.md << 'EOF'
你现在是 UADK 代码审查员。请严格按照 .tff/skills/code-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审查清单，参考 docs/uadk-reference.md 第6节和第9节。
EOF

cat > .claude/commands/uadk-build.md << 'EOF'
你现在是构建部署器。请严格按照 .tff/skills/build-deployer/SKILL.md 中的定义工作。首先阅读该文件获取服务器规则和构建流程，注意必须向用户确认服务器信息。
EOF

cat > .claude/commands/uadk-test.md << 'EOF'
你现在是测试调试器。请严格按照 .tff/skills/tester-debugger/SKILL.md 中的定义工作。首先阅读该文件获取测试矩阵、BD dump 和 GDB 调试规则。
EOF

cat > .claude/commands/uadk-docs.md << 'EOF'
你现在是技术文档编写者。请严格按照 .tff/skills/technical-writer/SKILL.md 中的定义工作。首先阅读该文件获取提交信息和 PR 描述规则。
EOF

cat > .claude/commands/uadk-meta.md << 'EOF'
你现在是元审查器。请严格按照 .tff/skills/meta-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审计维度和检查清单。
EOF
```

配置后路径 B 简化为（在 Claude Code 交互会话中输入）：

```
/uadk-analyze   → 描述需求
/uadk-develop   → 描述要改什么
/uadk-review    → 粘贴 diff
/uadk-build     → 提供服务器信息
/uadk-test      → 确认测试范围
/uadk-docs      → 生成 commit message
```

---

## 路径对比

| 维度 | 路径 A (tff-cc) | 路径 B (直调) |
|------|----------------|-------------|
| **启动方式** | `/tff-cc:new → discuss → plan → execute → verify → ship → learn` | 手动 Read skill → 切换角色 |
| **状态管理** | SQLite 自动跟踪 slice 状态和依赖 | 你自己记住进度 |
| **上下文传递** | tff-cc 自动将上游输出注入下游 agent | 你需要手动复制/引用上一步结果 |
| **并行执行** | 同 wave 的 slice 自动并行 | 你手动开多个会话 |
| **质量门禁** | verify 不过自动阻塞后续 execute | 你自行判断是否通过 |
| **学习改进** | `/tff-cc:learn` 自动审计并建议 skill 更新 | 需要你意识到问题才能改进 |
| **适合场景** | 多文件、多功能、团队协作的大型项目 | 单文件修改、快速修复、探索性开发 |
| **学习曲线** | 需要安装配置 tff-cc | 零配置，直接开始 |
