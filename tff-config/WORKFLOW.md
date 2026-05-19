# UADK 多 Agent 开发工作流

## 概述

本文档描述如何使用 7-agent UADK 开发工作流与 tff-cc 编排。

## 前置条件

- Node.js 20+ 已安装
- tff-cc 已安装：`npm install -g @the-forge-flow/tff-cc`
- Claude Code 已安装 tff-cc 插件（可选但推荐）
- 鲲鹏服务器访问：`ssh root@192.168.90.141`

## 配置

### 1. 安装 tff-cc

```bash
npm install -g @the-forge-flow/tff-cc

# 验证
tff-tools --version
```

### 2. 安装 Claude Code 插件（可选）

```bash
claude /plugin marketplace add MonsieurBarti/tff-mono
claude /plugin install tff-cc@the-forge-flow
# 验证：/tff:help
```

### 3. 配置项目

```bash
# 复制配置
cp tff-config/settings.yaml .tff/settings.yaml
```

## 工作流：添加 UADK 功能

### 阶段 1：任务分析
```bash
/tff:discuss
# 任务分析器 agent 分解需求
# 输出：带依赖关系和 agent 分配的任务树
```

### 阶段 2：代码开发
```bash
/tff:execute <slice-id>
# UADK 开发者 agent 实现变更
# 波次 0：代码变更
# 波次 1：构建和测试
```

### 阶段 3：代码审查
```bash
/tff:verify <slice-id>
# 代码审查员 agent 检查：
# - SQE 字段正确性（位位置、掩码）
# - 缓冲区安全（边界、溢出）
# - 硬件错误处理
# - 编码风格合规
```

### 阶段 4：构建与部署
```bash
# 构建部署器 agent：
# 1. 在远程服务器编译：cd /root/hch/lz4_uadk && make
# 2. 加载驱动：modprobe hisi_zip uacce_mode=1 perf_mode=1
# 3. 验证设备：cat /sys/class/uacce/hisi_zip-0/available_instances
```

### 阶段 5：测试与调试
```bash
# 测试调试器 agent：
# 1. 运行往返测试：./test_lz4_uadk
# 2. 需要时启用 BD dump：WD_BD_DUMP=1 ./lz4_debug ...
# 3. 性能剖析：perf record -g ./bench_lz4_uadk ...
# 4. 收集错误数据并路由回开发者
```

### 阶段 6：文档编写
```bash
# 技术文档编写者 agent：
# 1. 更新 README 添加新功能
# 2. 撰写提交信息（不含 "claude"/"AI"/"anthropic"）
# 3. 创建 PR 描述
# 4. 更新变更日志
```

### 阶段 7：元审查
```bash
# 会话结束后，元审查器 agent：
# 1. 审计 agent 对其 skill 的遵循度
# 2. 识别 skill 缺口
# 3. 提出 skill 改进建议
# 运行：/tff:learn
```

## 环境变量

| 变量 | 用途 | 默认值 |
|------|------|--------|
| WD_BD_DUMP | BD/SQE dump 级别（0=关, 1=关键字段, 2=完整） | 0 |
| WD_COMP_EPOLL_EN | 启用 epoll 模式 | 0 |
| LZ4_UADK_HW_CONCURRENCY | 硬件并发数 | 8 |
| LZ4_UADK_QUIET | 关闭调试输出 | 0 |

## 服务器信息

| 属性 | 值 |
|------|-----|
| IP | 192.168.90.141 |
| BMC | 192.168.90.140 |
| 架构 | aarch64（鲲鹏 920，128 核） |
| 操作系统 | openEuler |
| UADK 源码 | /root/hch/uadk/ |
| LZ4 UADK | /root/hch/lz4_uadk/ |
