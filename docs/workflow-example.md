# UADK 多 Agent 开发工作流 — 完整示例

> 以"为 hisi_zip 驱动新增 lz4 压缩算法硬件卸载支持"为例，演示两种使用路径的完整流程。

## 路径选择

| 路径 | 编排方式 | 适合场景 |
|------|---------|---------|
| **A. tff-cc 编排** | `/tff:discuss → execute → verify → learn` | 大型功能，需要严格 agent 流水线和质量门禁 |
| **B. 直调 Agent** | 手动按顺序加载各 agent 的 SKILL.md | 中小任务，快速迭代，零配置 |

---

## 路径 A：tff-cc 全自动编排

### 原理

tff-cc 是编排器，读取 `tff-config/settings.yaml` 中的工作流定义，按阶段依次调度 7 个 agent。状态通过 SQLite 持久化，上游输出自动作为下游的上下文。

```
你的需求
   │
   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ /tff:discuss│ ──► │/tff:execute │ ──► │ /tff:verify │
│  任务分析   │     │  代码开发   │     │  代码审查   │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
   ┌─────────────┐     ┌─────────────┐        │
   │ /tff:learn  │ ◄── │/tff:execute │ ◄──────┘
   │  元审查     │     │ 构建+测试   │
   └─────────────┘     └─────────────┘
```

### 阶段 0：环境准备

```bash
# 1. 安装 tff-cc
npm install -g @the-forge-flow/tff-cc

# 2. 安装 Claude Code 插件（可选但推荐）
claude /plugin marketplace add MonsieurBarti/tff-mono
claude /plugin install tff-cc@the-forge-flow

# 3. 配置项目
cp tff-config/settings.yaml .tff/settings.yaml

# 4. 验证
/tff:help
# 输出：
#   /tff:discuss  — 任务分析
#   /tff:execute  — 执行开发
#   /tff:verify   — 代码审查
#   /tff:learn    — 元审查与学习
```

### 阶段 1：`/tff:discuss` — 需求分解

在 Claude Code 中输入：

```
/tff:discuss

需求：为 hisi_zip 驱动新增 lz4 压缩算法硬件卸载支持。
约束：
- lz4: MINMATCH=4, LASTLITERALS=12, max_offset=65535
- 输出缓冲区 ≥ 2× 输入（min 8192 bytes）
- 参考 lz4_uadk 应用层的实现思路
- 不能破坏现有硬件结构体 ABI
```

**tff-cc 做的事情**：启动 `task-analyzer` agent，加载 `.tff/skills/task-analyzer/SKILL.md` 和 `docs/uadk-reference.md`，分解需求为原子任务，存入 SQLite。

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

### 阶段 2：`/tff:execute` — 逐 Slice 执行

tff-cc 按依赖拓扑顺序调度 agent。每个 slice 完成后自动标记状态，下游 slice 就绪后才能执行。

```bash
# Slice #1：添加枚举（无依赖，立即就绪）
/tff:execute 1
```

tff-cc 启动 `uadk-developer` agent，从 SQLite 读取上下文，执行代码变更：

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

```bash
# Slice #1 完成，Slice #2 自动就绪
/tff:execute 2
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

### 阶段 3：`/tff:verify` — 代码审查

```bash
/tff:verify 3
```

tff-cc 启动 `code-reviewer` agent，传入 Slice #1、#2 的 diff：

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

### 阶段 4：`/tff:execute` — 构建 + 测试

审查通过后继续：

```bash
/tff:execute 4   # 构建部署
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

```bash
/tff:execute 5   # 测试
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

### 阶段 5：文档合入

```bash
/tff:execute 6   # 提交信息
```

```
feat: add lz4 hardware offload support to hisi_zip driver

Register lz4 algorithm in hisi_comp ops table with fill_buf_lz4
SQE filling function. LZ4 constraints enforced: MINMATCH=4,
LASTLITERALS=12, max_offset=65535, output buffer >= 2x input.

Signed-off-by: Developer <dev@example.com>
```

### 阶段 6：`/tff:learn` — 元审查

```bash
/tff:learn
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

在 `.claude/settings.json` 中注册快捷命令，一个 `/` 即可切换 agent：

```json
{
  "slashCommands": [
    {
      "name": "/uadk-analyze",
      "prompt": "你现在是任务分析器。请严格按照 .tff/skills/task-analyzer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则和记忆，同时参考 docs/uadk-reference.md。"
    },
    {
      "name": "/uadk-develop",
      "prompt": "你现在是 UADK 开发者。请严格按照 .tff/skills/uadk-developer/SKILL.md 中的定义工作。首先阅读该文件获取完整规则、SQE 参考和代码模式，参考 docs/uadk-reference.md 第5-7节和第9-10节。"
    },
    {
      "name": "/uadk-review",
      "prompt": "你现在是 UADK 代码审查员。请严格按照 .tff/skills/code-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审查清单，参考 docs/uadk-reference.md 第6节和第9节。"
    },
    {
      "name": "/uadk-build",
      "prompt": "你现在是构建部署器。请严格按照 .tff/skills/build-deployer/SKILL.md 中的定义工作。首先阅读该文件获取服务器规则和构建流程，注意必须向用户确认服务器信息。"
    },
    {
      "name": "/uadk-test",
      "prompt": "你现在是测试调试器。请严格按照 .tff/skills/tester-debugger/SKILL.md 中的定义工作。首先阅读该文件获取测试矩阵、BD dump 和 GDB 调试规则。"
    },
    {
      "name": "/uadk-docs",
      "prompt": "你现在是技术文档编写者。请严格按照 .tff/skills/technical-writer/SKILL.md 中的定义工作。首先阅读该文件获取提交信息和 PR 描述规则。"
    },
    {
      "name": "/uadk-meta",
      "prompt": "你现在是元审查器。请严格按照 .tff/skills/meta-reviewer/SKILL.md 中的定义工作。首先阅读该文件获取审计维度和检查清单。"
    }
  ]
}
```

配置后路径 B 简化为：

```bash
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
| **启动方式** | `/tff:discuss → execute → verify → learn` | 手动 Read skill → 切换角色 |
| **状态管理** | SQLite 自动跟踪 slice 状态和依赖 | 你自己记住进度 |
| **上下文传递** | tff-cc 自动将上游输出注入下游 agent | 你需要手动复制/引用上一步结果 |
| **并行执行** | 同 wave 的 slice 自动并行 | 你手动开多个会话 |
| **质量门禁** | verify 不过自动阻塞后续 execute | 你自行判断是否通过 |
| **学习改进** | `/tff:learn` 自动审计并建议 skill 更新 | 需要你意识到问题才能改进 |
| **适合场景** | 多文件、多功能、团队协作的大型项目 | 单文件修改、快速修复、探索性开发 |
| **学习曲线** | 需要安装配置 tff-cc | 零配置，直接开始 |
