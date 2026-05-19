# UADK 全面技术参考

> 基于 https://github.com/Linaro/uadk 源码和设计文档编写。
> 版本对应 UADK v2.10+，Apache 2.0 许可证。

---

## 目录

1. [架构概览](#1-架构概览)
2. [项目结构](#2-项目结构)
3. [构建系统](#3-构建系统)
4. [公共 API 参考](#4-公共-api-参考)
5. [压缩驱动详解](#5-压缩驱动详解)
6. [SQE 硬件描述符](#6-sqe-硬件描述符)
7. [队列管理](#7-队列管理)
8. [其他硬件驱动](#8-其他硬件驱动)
9. [内存管理](#9-内存管理)
10. [SVA 与 No-SVA 模式](#10-sva-与-no-sva-模式)
11. [NUMA 拓扑](#11-numa-拓扑)
12. [环境变量配置](#12-环境变量配置)
13. [配置文件 uadk.cnf](#13-配置文件-uadkcnf)
14. [驱动动态加载](#14-驱动动态加载)
15. [调试与诊断](#15-调试与诊断)
16. [版本历史](#16-版本历史)

---

## 1. 架构概览

### 1.1 分层架构

```
┌─────────────────────────────────────┐
│          用户应用程序                  │
│    (直接使用 libwd 或算法库)           │
├─────────────────────────────────────┤
│  算法库层 (libwd_comp / libwd_crypto) │
│  - 会话管理                           │
│  - 调度器 (scheduler)                 │
│  - 环境变量解析                       │
├─────────────────────────────────────┤
│  libwd 基础层                         │
│  - 上下文 (context) 管理              │
│  - 内存映射 (mmap)                    │
│  - 设备发现                           │
│  - 内存池 (mempool/blkpool)           │
├─────────────────────────────────────┤
│  UACCE 内核接口 (/dev/hisi_zip-*)    │
│  - 字符设备                           │
│  - sysfs 信息导出                     │
│  - ioctl / mmap                       │
├─────────────────────────────────────┤
│  硬件加速器 (鲲鹏 hisi_zip/hisi_sec)  │
└─────────────────────────────────────┘
```

### 1.2 核心概念

| 概念 | 说明 |
|------|------|
| **Context (上下文)** | CPU 与硬件加速器之间的双向通信资源。每次 `open(/dev/hisi_zip-*)` 创建一个 context |
| **Session (会话)** | 上下文的超集，封装算法类型、压缩级别、窗口大小、操作模式。用户通过 handle_t 引用 |
| **QP (Queue Pair)** | 硬件队列对。包含 SQ（提交队列）和 CQ（完成队列）|
| **SQE (Submission Queue Entry)** | 提交队列条目，32 双字硬件命令描述符 |
| **SVA** | Shared Virtual Addressing，IOMMU 支持的共享虚拟地址 |
| **DUS** | Device User Share region，设备用户共享内存区域 |
| **MMIO** | Memory-Mapped I/O，设备 MMIO 寄存器区域 |

---

## 2. 项目结构

```
uadk/
├── drv/                    # 硬件驱动（用户态）
│   ├── hisi_comp.c         # ZIP 压缩驱动 (1920行)
│   ├── hisi_comp_huf.c     # Huffman 编码辅助 (6452B)
│   ├── hisi_qm_udrv.c      # 队列管理通用层 (1168行)
│   ├── hisi_qm_udrv.h      # 队列管理头文件
│   ├── hisi_sec.c          # SEC 安全引擎驱动 (4017行)
│   ├── hisi_hpre.c         # HPRE 高性能RSA驱动 (3039行)
│   ├── hisi_dae.c          # DAE 数据加速引擎 (1335行)
│   ├── hisi_udma.c         # UDMA 驱动
│   ├── isa_ce_sm3.c        # ARM CE SM3 指令加速
│   ├── isa_ce_sm4.c        # ARM CE SM4 指令加速
│   └── hash_mb/            # SVE 多缓冲区哈希
├── include/                # 公共头文件
│   ├── wd.h                # 核心 API (641行)
│   ├── wd_comp.h           # 压缩 API (267行)
│   ├── wd_alg_common.h     # 算法通用结构体
│   ├── wd_util.h           # 工具函数
│   ├── wd_cipher.h         # 对称加密 API
│   ├── wd_digest.h         # 摘要 API
│   ├── wd_aead.h           # 认证加密 API
│   ├── wd_ecc.h            # 椭圆曲线 API
│   ├── wd_rsa.h            # RSA API
│   ├── wd_dh.h             # DH API
│   ├── wd_dae.h            # DAE API
│   ├── uacce.h             # UACCE 内核接口定义
│   └── drv/                # 驱动内部头文件
│       └── wd_comp_drv.h   # 压缩驱动消息结构体
├── lib/                    # pkg-config 模板
├── wd.c                    # 核心库实现
├── wd_comp.c               # 压缩算法库 (69697B)
├── wd_util.c               # 工具函数 (69697B)
├── wd_alg.c                # 算法注册/发现
├── wd_sched.c              # 调度器实现
├── wd_mempool.c            # 内存池管理
├── v1/                     # v1 API（兼容旧版）
├── test/                   # 测试用例
├── sample/                 # 使用示例
├── uadk_tool/              # 性能基准测试工具
├── docs/                   # 设计文档
├── configure.ac            # autotools 配置
├── Makefile.am             # 顶层 Makefile
├── uadk.cnf                # 驱动加载配置
└── libwd*.map              # 符号导出映射
```

---

## 3. 构建系统

### 3.1 构建命令

```bash
# 标准构建
./autogen.sh
./configure
make -j$(nproc)
sudo make install

# 调试构建
CFLAGS="-g -O0" ./configure
make -j$(nproc)

# 启用调试日志
./configure --enable-debug-log

# 静态链接驱动
./configure --with-static_drv

# 编译为静态库
./configure --with-static_drv
```

### 3.2 构建产物

| 产物 | 说明 |
|------|------|
| `libwd.so` | 核心库，提供 context/mmio/mempool |
| `libwd_comp.so` | 压缩算法库，提供压缩/解压 API |
| `libwd_crypto.so` | 加密算法库，提供加密/解密 API |
| `libhisi_zip.so` | 鲲鹏 ZIP 用户态驱动 |
| `libhisi_sec.so` | 鲲鹏 SEC 用户态驱动 |
| `libhisi_hpre.so` | 鲲鹏 HPRE 用户态驱动 |
| `libisa_ce.so` | ARM CE 指令加速驱动 |
| `uadk_tool` | 性能基准测试工具 |

### 3.3 编译要求

- `-Wall -Wextra` 强制，零警告
- aarch64 架构（鲲鹏 920 或兼容平台）
- 依赖：libnuma、libpthread、libdl、openssl（可选）、zlib（可选）

---

## 4. 公共 API 参考

### 4.1 核心 API (`wd.h`)

#### 设备发现

```c
// 获取支持某算法的设备列表
struct uacce_dev_list *wd_get_accel_list(const char *alg_name);

// 获取距离当前 NUMA node 最近的设备
struct uacce_dev *wd_get_accel_dev(const char *alg_name);

// 按 NUMA ID 查找有最多可用 ctx 的设备
struct uacce_dev *wd_find_dev_by_numa(struct uacce_dev_list *list, int numa_id);
```

#### Context 管理

```c
// 请求一个通信上下文
handle_t wd_request_ctx(struct uacce_dev *dev);

// 释放上下文
void wd_release_ctx(handle_t h_ctx);

// 强制释放（立即释放硬件资源）
int wd_release_ctx_force(handle_t h_ctx);

// 启动上下文（获取 MMIO/DUS）
int wd_ctx_start(handle_t h_ctx);

// 获取可用 context 数量
int wd_get_avail_ctx(struct uacce_dev *dev);
```

#### 内存映射

```c
// mmap 设备区域到用户空间
void *wd_ctx_mmap_qfr(handle_t h_ctx, enum uacce_qfrt qfrt);
// qfrt: UACCE_QFRT_MMIO (MMIO寄存器) | UACCE_QFRT_DUS (设备用户共享区域)

// 取消映射
void wd_ctx_unmap_qfr(handle_t h_ctx, enum uacce_qfrt qfrt);
```

#### 设备信息查询

```c
char *wd_get_accel_name(char *dev_path, int no_apdx);
int wd_get_numa_id(handle_t h_ctx);
char *wd_ctx_get_dev_name(handle_t h_ctx);
int wd_is_sva(handle_t h_ctx);
int wd_is_isolate(struct uacce_dev *dev);
```

### 4.2 压缩 API (`wd_comp.h`)

#### 算法类型枚举

```c
enum wd_comp_alg_type {
    WD_DEFLATE,    // 原始 DEFLATE
    WD_ZLIB,       // zlib 包装的 DEFLATE
    WD_GZIP,       // gzip 包装的 DEFLATE
    WD_LZ77_ZSTD,  // LZ77 + ZSTD 编码
    WD_LZ4,        // LZ4 压缩 (v2.10+)
    WD_LZ77_ONLY,  // 纯 LZ77 (v2.10+)
    WD_COMP_ALG_MAX,
};
```

#### 初始化接口

```c
// 标准初始化（用户管理 ctx 和 scheduler）
int wd_comp_init(struct wd_ctx_config *config, struct wd_sched *sched);

// 简化初始化（v2 接口，自动管理资源）
int wd_comp_init2_(char *alg, __u32 sched_type, int task_type,
                   struct wd_ctx_params *ctx_params);
#define wd_comp_init2(alg, sched_type, task_type) \
    wd_comp_init2_(alg, sched_type, task_type, NULL)

// 环境变量初始化
int wd_comp_env_init(struct wd_sched *sched);

// 反初始化
void wd_comp_uninit(void);
void wd_comp_uninit2(void);
void wd_comp_env_uninit(void);
```

#### 会话管理

```c
struct wd_comp_sess_setup {
    enum wd_comp_alg_type alg_type; // 算法类型
    enum wd_comp_level comp_lv;     // 压缩级别 1-15
    enum wd_comp_winsz_type win_sz; // 窗口大小
    enum wd_comp_op_type op_type;   // 压缩 or 解压
    void *sched_param;
    struct wd_mm_ops mm_ops;        // 自定义内存操作
    enum wd_mem_type mm_type;       // 内存类型
};

handle_t wd_comp_alloc_sess(struct wd_comp_sess_setup *setup);
void wd_comp_free_sess(handle_t h_sess);
int wd_comp_reset_sess(handle_t h_sess);  // 流模式重置
```

#### 压缩/解压请求

```c
struct wd_comp_req {
    union { void *src; struct wd_datalist *list_src; };
    __u32 src_len;
    union { void *dst; struct wd_datalist *list_dst; };
    __u32 dst_len;
    wd_alg_comp_cb_t *cb;        // 异步回调
    void *cb_param;
    enum wd_comp_op_type op_type;
    enum wd_buff_type data_fmt;   // WD_FLAT_BUF / WD_SGL_BUF
    __u32 last;                   // 流模式最后一块标志
    __u32 status;                 // 返回状态
    void *priv;                   // 算法私有数据
};
```

#### 操作模式

```c
// 同步块模式
int wd_do_comp_sync(handle_t h_sess, struct wd_comp_req *req);

// 同步流模式
int wd_do_comp_strm(handle_t h_sess, struct wd_comp_req *req);

// 同步流模式 v2（自动拆分大数据块）
int wd_do_comp_sync2(handle_t h_sess, struct wd_comp_req *req);

// 异步模式
int wd_do_comp_async(handle_t h_sess, struct wd_comp_req *req);

// 异步轮询
int wd_comp_poll(__u32 expt, __u32 *count);
int wd_comp_poll_ctx(__u32 idx, __u32 expt, __u32 *count);
```

### 4.3 调度器 (`wd_alg_common.h`)

```c
struct wd_sched {
    const char *name;
    int sched_policy;
    handle_t (*sched_init)(handle_t h_sched_ctx, void *sched_param);
    __u32 (*pick_next_ctx)(handle_t h_sched_ctx, void *sched_key,
                           const int sched_mode);
    int (*poll_policy)(handle_t h_sched_ctx, __u32 expect, __u32 *count);
    handle_t h_sched_ctx;
};
```

UADK 提供默认的 `sample_sched_alloc()` / `sample_sched_release()` 实现。

---

## 5. 压缩驱动详解

### 5.1 驱动入口

文件：`drv/hisi_comp.c`（1920 行）

hisi_zip 驱动注册了 6 个算法驱动实例：

```c
static struct wd_alg_driver zip_alg_driver[] = {
    GEN_ZIP_ALG_DRIVER("zlib"),
    GEN_ZIP_ALG_DRIVER("gzip"),
    GEN_ZIP_ALG_DRIVER("deflate"),
    GEN_ZIP_ALG_DRIVER("lz77_zstd"),
    GEN_ZIP_ALG_DRIVER("lz4"),
    GEN_ZIP_ALG_DRIVER("lz77_only")
};
```

每个驱动实例包含：
- `init` / `exit`：硬件初始化/清理
- `send`：`hisi_zip_comp_send()` — 填充 SQE 并发送到硬件
- `recv`：`hisi_zip_comp_recv()` — 从硬件接收结果并解析
- `get_usage`：查询设备带宽使用率

### 5.2 算法枚举（硬件层）

```c
enum alg_type {
    HW_DEFLATE = 0x1,       // 标准 DEFLATE
    HW_ZLIB,                // zlib
    HW_GZIP,                // gzip
    HW_LZ4,                 // LZ4
    HW_LZ77_ZSTD_PRICE = 0x42,  // LZ77+ZSTD (带价格优化)
    HW_LZ77_ZSTD,           // LZ77+ZSTD
    HW_LZ77_ONLY = 0x40,    // 纯 LZ77
    HW_LZ77_ONLY_PRICE,     // 纯 LZ77 (带价格优化)
};
```

### 5.3 算法操作函数表

每个算法注册一组操作函数（`struct hisi_zip_sqe_ops`）：

```c
struct hisi_zip_sqe_ops {
    const char *alg_name;
    int (*fill_buf[2])(...);       // [0]=flat, [1]=sgl
    void (*fill_sqe_type)(...);    // 设置 SQE type (V1/V3)
    void (*fill_alg)(...);         // 设置算法类型字段
    void (*fill_tag)(...);         // 设置 tag
    int (*fill_comp_level)(...);  // 设置压缩级别
    void (*get_data_size)(...);   // 从 SQE 提取结果
    int (*get_tag)(...);           // 从 SQE 提取 tag
};
```

### 5.4 各算法的缓冲区要求

| 算法 | 最大输入 | 最小输出缓冲区 | 备注 |
|------|----------|---------------|------|
| deflate/zlib/gzip | 8MB (`HZ_MAX_SIZE`) | `in_size × 1.125` | 最小输出 200 字节（`SW_STOREBUF_TH`） |
| lz4 | 8MB | `2 × in_size`（最小 8192） | 仅支持压缩，块模式 |
| lz77_zstd | 128KB (`ZSTD_MAX_SIZE`) | `in_size + 16 + 784` (频率数据) | L9 价格模式需额外 ~4096 字节 |
| lz77_only | ~8MB | `in_size + 16 + 32` (DFX 数据) | L9 价格模式需额外 ~4096 字节 |

### 5.5 流模式（Stateful）缓冲管理

流模式压缩场景下，当输出缓冲区小于 200 字节时，硬件无法直接写入用户缓冲区。此时使用内部存储缓冲区（`CTX_STOREBUF_OFFSET = 0x9800`）暂存数据。

相关的 store_buf 逻辑在 `check_enable_store_buf()` 和 `copy_to_out()` 中实现。

---

## 6. SQE 硬件描述符

### 6.1 结构体定义

文件：`drv/hisi_comp.c:170-232`

```c
struct hisi_zip_sqe {
    __u32 consumed;           // dw0:  硬件消耗的输入字节数
    __u32 produced;           // dw1:  硬件产生的输出字节数
    __u32 comp_data_length;   // dw2:  压缩数据长度
    __u32 dw3;                // dw3:  status(位0-7), rsvd(位8-31)
    __u32 input_data_length;  // dw4:  输入数据大小
    __u32 dw5;                // dw5:  保留
    __u32 dw6;                // dw6:  保留
    __u32 dw7;                // dw7:  in_sge_offset(0-23), flush(25),
                              //       stream_mode(26), stream_new(27),
                              //       sqe_type(28-31)
    __u32 dw8;                // dw8:  out_sge_offset(0-23)
    __u32 dw9;                // dw9:  request_type(0-7),
                              //       buffer_type(8-11),
                              //       window_size(12-15)
    __u32 dw10-dw12;          // 保留
    __u32 dw13;               // tag (仅 SQE type 0)
    __u32 dest_avail_out;     // dw14: 输出缓冲区容量
    __u32 ctx_dw0;            // dw15: 上下文状态（流模式）
    __u32 dw16-dw17;          // 保留
    __u32 source_addr_l;      // dw18: 源地址低 32 位
    __u32 source_addr_h;      // dw19: 源地址高 32 位
    __u32 dest_addr_l;        // dw20: 目标地址低 32 位
    __u32 dest_addr_h;        // dw21: 目标地址高 32 位
    __u32 stream_ctx_addr_l;  // dw22: 流上下文地址低 32 位
    __u32 stream_ctx_addr_h;  // dw23: 流上下文地址高 32 位
    __u32 literals_addr_l;    // dw24: 字面量地址低 32 位 (lz77)
    __u32 literals_addr_h;    // dw25: 字面量地址高 32 位 (lz77)
    __u32 dw26;               // tag (SQE type 3)
    __u32 dw27;               // 保留
    __u32 ctx_dw1;            // dw28: 上下文状态（流模式）
    __u32 ctx_dw2;            // dw29: 上下文状态（流模式）
    __u32 isize;              // dw30: gzip 原始大小
    __u32 dw31;               // dw31: checksum(gzip) 或
                              //       litlen_overflow_pos(0-23)
                              //       litlen_overflow_cnt(24-31)
};
```

### 6.2 关键位字段速查

| 字段 | 位置 | 掩码/偏移 | 说明 |
|------|------|-----------|------|
| status | dw3:0-7 | `0xff` | 硬件执行状态 |
| sqe_type | dw7:28-31 | `>>28` | V1=`0x0`, V3=`0x3` |
| flush_type | dw7:25 | `>>25 & 1` | 0=sync_flush, 1=finish |
| stream_mode | dw7:26 | `>>26 & 1` | 0=stateless, 1=stateful |
| stream_new | dw7:27 | `>>27 & 1` | 0=old_stream, 1=new_stream |
| in_sge_offset | dw7:0-23 | `0xffffff` | SGL 模式下的输入偏移 |
| request_type | dw9:0-7 | `0xff` | 请求类型（压缩/解压） |
| buffer_type | dw9:8-11 | `>>8 & 0xf` | 0=flat, 1=sgl |
| window_size | dw9:12-15 | `>>12 & 0xf` | 窗口大小 |
| litlen_overflow_pos | dw31:0-23 | `0xffffff` | 字面量长度溢出位置 |
| litlen_overflow_cnt | dw31:24-31 | `>>24` | 字面量长度溢出计数 |

### 6.3 硬件错误码

| 错误码 | 宏定义 | 含义 |
|--------|--------|------|
| `0x0` | — | 成功 |
| `0x01` | `HZ_DECOMP_NO_SPACE` | 解压输出空间不足 |
| `0x02` | `HZ_DECOMPING_NO_SPACE` | 解压进行中空间不足 |
| `0x03` | `HZ_DECOMP_BLK_NOSTART` | 解压块未开始（需继续发送请求） |
| `0x0d` | `HZ_NEGACOMPRESS` | 负压缩（输出 ≥ 输入）— 正常行为 |
| `0x0e` | `ERR_DSTLEN_OUT` | 输出缓冲区严重不足 — 最常见错误 |
| `0x10` | `HZ_CRC_ERR` | CRC 校验失败 |
| `0x13` | `HZ_DECOMP_END` | 解压流结束 — 正常流结束标记 |

### 6.4 SQE 类型 (V1 vs V3)

| 类型 | 值 | Tag 位置 | 适用算法 |
|------|-----|---------|----------|
| V1 | `0x0`（dw7:28-31） | dw13 | deflate、zlib、gzip、lz4 |
| V3 | `0x30000000`（dw7:28-31） | dw26 | lz77_zstd、lz77_only |

### 6.5 物理地址映射

SVA 模式下直接使用虚拟地址（硬件通过 IOMMU 转换）：
```c
sqe->source_addr_l = lower_32_bits(addr->src_addr);
sqe->source_addr_h = upper_32_bits(addr->src_addr);
```

No-SVA 模式下需要 `zip_mem_map()` 获取物理地址（iova）。

---

## 7. 队列管理

### 7.1 QP 结构体

```c
struct hisi_qp {
    struct hisi_qm_queue_info q_info;  // 队列硬件信息
    handle_t h_sgl_pool;               // SGL 池句柄
    handle_t h_ctx;                    // 关联的 context
    void *priv;                        // 驱动私有数据
};
```

### 7.2 队列信息

```c
struct hisi_qm_queue_info {
    void *sq_base;          // 提交队列基地址
    void *cq_base;          // 完成队列基地址
    int sqe_size;           // SQE 大小（字节）
    void *mmio_base;        // MMIO 基地址
    void *db_base;          // 门铃 (doorbell) 基地址
    __u16 sq_tail_index;    // SQ 尾指针
    __u16 cq_head_index;    // CQ 头指针
    __u16 sq_depth;         // SQ 深度
    __u16 cq_depth;         // CQ 深度
    bool cqc_phase;         // CQ 完成阶段标志
    pthread_spinlock_t sd_lock;  // 发送锁
    pthread_spinlock_t rv_lock;  // 接收锁
};
```

### 7.3 关键 API

```c
// 发送 SQE 到硬件队列
int hisi_qm_send(handle_t h_qp, const void *req, __u16 expect, __u16 *count);

// 从硬件队列接收完成条目
int hisi_qm_recv(handle_t h_qp, void *resp, __u16 expect, __u16 *count);

// 分配/释放 QP
handle_t hisi_qm_alloc_qp(struct hisi_qm_priv *config, handle_t ctx);
void hisi_qm_free_qp(handle_t h_qp);

// SGL 池管理
handle_t hisi_qm_create_sglpool(__u32 sgl_num, __u32 sge_num, struct wd_mm_ops *mm_ops);
void hisi_qm_destroy_sglpool(handle_t sgl_pool);
void *hisi_qm_get_hw_sgl(handle_t sgl_pool, struct wd_datalist *sgl);
void hisi_qm_put_hw_sgl(handle_t sgl_pool, void *hw_sgl);
```

### 7.4 门铃机制

发送方写完 SQE 后通过门铃通知硬件；硬件完成后更新 CQE（完成队列条目）并通过相位翻转机制通知软件。

---

## 8. 其他硬件驱动

### 8.1 hisi_sec（安全引擎）

- 文件：`drv/hisi_sec.c`（4017 行）
- 功能：对称加密（AES/SM4/DES/3DES）、摘要（SHA/SM3/MD5）、认证加密（GCM/CCM）
- 支持算法：ECB/CBC/CTR/XTS/OFB/CFB 模式

### 8.2 hisi_hpre（高性能 RSA 引擎）

- 文件：`drv/hisi_hpre.c`（3039 行）
- 功能：RSA、DH、ECDH、ECDSA、SM2、X25519、X448

### 8.3 hisi_dae（数据加速引擎，v2.8+）

- 文件：`drv/hisi_dae.c`（1335 行）
- 功能：数据搬移、hash-agg（SUM/COUNT/MIN/MAX）、hash-join

### 8.4 ISA 指令加速

- `isa_ce_sm3.c/h`：ARMv8 Crypto Extension 实现的 SM3
- `isa_ce_sm4.c/h`：ARMv8 Crypto Extension 实现的 SM4
- `hash_mb/`：SVE 多缓冲区哈希（SM3、MD5）

---

## 9. 内存管理

### 9.1 内存操作接口

```c
struct wd_mm_ops {
    wd_alloc alloc;           // 内存分配
    wd_free free;             // 内存释放
    wd_map iova_map;          // VA → DMA 地址映射
    wd_unmap iova_unmap;      // 取消映射
    wd_bufsize get_bufsize;   // 获取缓冲区大小（可选）
    void *usr;                // 用户自定义数据
    bool sva_mode;            // 是否 SVA 模式
};
```

### 9.2 内存池

```c
// 创建/销毁内存池
handle_t wd_mempool_create(size_t size, int node);
void wd_mempool_destroy(handle_t mempool);

// 从内存池分配块
handle_t wd_blockpool_create(handle_t mempool, size_t block_size, size_t block_num);
void *wd_block_alloc(handle_t blkpool);
void wd_block_free(handle_t blkpool, void *addr);
```

### 9.3 内存类型

```c
enum wd_mem_type {
    UADK_MEM_AUTO,   // 自动选择
    UADK_MEM_USER,   // 用户管理内存
    UADK_MEM_PROXY,  // 代理内存（No-SVA）
};
```

---

## 10. SVA 与 No-SVA 模式

### 10.1 SVA 模式（默认）

- IOMMU 提供地址转换
- 用户态虚拟地址直接传给硬件
- 零拷贝（应用缓冲区和硬件共享同一物理页）
- 需求：IOMMU SVA 支持、Linux 5.10+

### 10.2 No-SVA 模式（v2.10+）

- 不支持 IOMMU SVA 的系统
- 需要通过 `iova_map()` 获取 DMA 地址
- 需要额外的 SGL 内存池（`h_nosva_sgl_pool`）
- 驱动内部处理地址映射/解映射

### 10.3 模式检测

```c
if (!mm_ops->sva_mode) {
    // No-SVA: 需要物理地址映射
    zip_mem_map(mm_ops, sqe, &addr);
} else {
    // SVA: 直接使用虚拟地址
    sqe->source_addr_l = lower_32_bits(addr.src_addr);
    sqe->source_addr_h = upper_32_bits(addr.src_addr);
}
```

---

## 11. NUMA 拓扑

### 11.1 设备分布

鲲鹏 920 通常有两个 hisi_zip 设备，分别位于不同的 NUMA node：

```bash
cat /sys/class/uacce/hisi_zip-0/node_id  # → 0
cat /sys/class/uacce/hisi_zip-1/node_id  # → 1（如果有两个设备）
```

### 11.2 NUMA 绑定规则

- CPU 和内存必须绑定到与 hisi_zip 设备相同的 NUMA node
- `ctx_set_num` 是 per-NUMA-node 分配，代码内部自动除以 `numa_num_configured_nodes()`
- 双设备并行测试需要禁用 NUMA 绑定：`numactl -n -1`

### 11.3 NUMA 感知 API

```c
int wd_get_numa_id(handle_t h_ctx);
struct uacce_dev *wd_find_dev_by_numa(struct uacce_dev_list *list, int numa_id);
struct uacce_dev *wd_get_accel_dev(const char *alg_name);  // 自动选择最近 NUMA
```

---

## 12. 环境变量配置

### 12.1 压缩算法环境变量

| 变量 | 格式 | 示例 |
|------|------|------|
| `WD_COMP_CTX_NUM` | `sync-comp:N@node,sync-decomp:M@node` | `sync-comp:10@0,async-decomp:20@1` |
| `WD_COMP_ASYNC_POLL_EN` | `0`/`1` | `1` 启用内部轮询线程 |
| `WD_COMP_ASYNC_POLL_NUM` | `N@node,M@node` | `2@0,4@2` |
| `WD_COMP_EPOLL_EN` | `0`/`1` | `1` 使能 epoll 等待 |

### 12.2 其他算法环境变量

类似格式，将 `COMP` 替换为 `CIPHER`、`AEAD`、`DIGEST`、`DH`、`RSA`、`ECC`：

```bash
WD_CIPHER_CTX_NUM=sync:10@0,async:20@1
WD_DIGEST_CTX_NUM=sync:10@0
```

### 12.3 配置策略

1. 如果一个 node 启用，所有 node 共享相同配置
2. 同类型多配置取最大 ctx 数
3. ctx 数为 0 的配置被忽略

---

## 13. 配置文件 uadk.cnf

文件：`uadk.cnf`（放置在与驱动 .so 相同的目录）

```
libhisi_zip.so
libhisi_sec.so
libhisi_hpre.so
libisa_ce.so
libisa_sve.so
libhisi_dae.so
libhisi_udma.so
```

- 每行一个驱动库名
- `#` 开头表示注释（不加载）
- 如果不指定不存在的 .so，会被跳过并记录日志
- 如果 uadk.cnf 不存在，使用默认加载方式

---

## 14. 驱动动态加载

### 14.1 V1 加载方式

```c
// 查找并加载 libhisi_zip.so
ret = wd_get_lib_file_path("libhisi_zip.so", lib_path, false);
wd_comp_setting.dlhandle = dlopen(lib_path, RTLD_NOW);
// __attribute__((constructor)) 自动调用 hisi_zip_probe()
```

### 14.2 V2 加载方式

```c
// 从目录批量加载所有驱动 .so
wd_comp_setting.dlh_list = wd_dlopen_drv(NULL);
```

### 14.3 静态链接

```bash
./configure --with-static_drv
```
编译时将驱动代码直接链接到算法库中，无需动态加载。

---

## 15. 调试与诊断

### 15.1 日志系统

```c
// 日志宏（默认输出到 syslog）
WD_DEBUG(fmt, args...)  // 调试日志
WD_INFO(fmt, args...)   // 信息日志
WD_ERR(fmt, args...)    // 错误日志
WD_DEV_ERR(h_ctx, fmt, args...)  // 带设备名的错误日志

// 当 WD_NO_LOG 定义时，改为 fprintf(stderr, ...)
```

### 15.2 设备信息查询

```bash
# 设备列表
ls /dev/hisi_zip-*

# 设备能力
cat /sys/class/uacce/hisi_zip-0/api          # API 版本
cat /sys/class/uacce/hisi_zip-0/algorithms   # 支持的算法
cat /sys/class/uacce/hisi_zip-0/available_instances  # 可用实例数
cat /sys/class/uacce/hisi_zip-0/node_id      # NUMA node
cat /sys/class/uacce/hisi_zip-0/flags         # SVA 标志

# 驱动参数
cat /sys/module/hisi_zip/parameters/uacce_mode
cat /sys/module/hisi_zip/parameters/perf_mode
cat /sys/module/hisi_zip/parameters/pf_q_num
```

### 15.3 SQE Dump（BD Dump）

通过 `WD_BD_DUMP` 环境变量控制：
- `0`：关闭（默认）
- `1`：打印关键字段（input_len、out_cap、consumed、produced、status、sqe_type、alg）
- `2`：完整 32 双字 hex dump

### 15.4 GDB 调试

```bash
# 调试构建
CFLAGS="-g -O0" ./configure && make

# 关键断点
break hisi_zip_comp_send   # SQE 提交前
break parse_zip_sqe        # 硬件返回后
break fill_buf_lz77        # LZ77 SQE 填充
break fill_buf_deflate     # deflate SQE 填充

# 检查 SQE
p/x *sqe                   # 按结构体打印
x/32wx sqe                 # 原始 32 双字 hex dump
```

### 15.5 性能剖析

```bash
# perf stat
perf stat -e cycles,instructions,cache-misses,branch-misses \
  LD_LIBRARY_PATH=/usr/local/lib uadk_tool benchmark --alg zlib ...

# perf record
perf record -g -F 99 LD_LIBRARY_PATH=/usr/local/lib ./bench_app
perf report
```

### 15.6 性能测试工具

```bash
uadk_tool benchmark --alg zlib --mode sva --opt 0 --sync \
  --pktlen 4096 --seconds 5 --thread 1 --multi 1 --ctxnum 1 --prefetch

uadk_tool benchmark --help  # 查看所有选项
```

---

## 16. 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|----------|
| v2.10 | 2025.12 | 新增 LZ4、LZ77_only、AEAD；No-SVA 模式全面支持；hash-agg/join/gather；uadk.cnf |
| v2.9.1 | 2025.07 | 修复 x86 构建 |
| v2.9 | 2025.06 | HMAC(SM3)-CBC(SM4)；ECC 高性能模式 |
| v2.8 | 2024.12 | DAE 加速器；hash-agg；uadk_tool 压缩测试 |
| v2.7 | 2024.06 | ARM CE SM3/SM4 指令加速；SVE SM3/MD5 多缓冲区 |
| v2.6 | 2023.12 | SM4-XTS；队列深度可配；AES-CTS |
| v2.5 | 2023.06 | init2 接口（简化初始化） |
| v2.4 | 2022.12 | 首个公开发布：加密+压缩+内存管理+调度器 |

---

## 附录：关键常量速查

```c
#define HZ_MAX_SIZE              (8 * 1024 * 1024)      // 8MB, 最大输入
#define ZSTD_MAX_SIZE            (1 << 17)              // 128KB, lz77_zstd 最大输入
#define HZ_SQE_TYPE_V1           0x0                    // SQE V1
#define HZ_SQE_TYPE_V3           0x30000000             // SQE V3
#define ERR_DSTLEN_OUT           0xe                    // 输出缓冲区不足
#define HZ_NEGACOMPRESS          0x0d                   // 负压缩
#define HZ_DECOMP_END            0x13                   // 解压结束
#define HZ_CRC_ERR               0x10                   // CRC 错误
#define min_out_buf_size(inl)    ((((__u64)(inl) * 9) + 7) >> 3)  // in*1.125
#define HW_CTX_SIZE              0x10000                // 64KB, 硬件上下文大小
#define CTX_STOREBUF_OFFSET      0x9800                 // 内部存储缓冲区偏移
#define SW_STOREBUF_TH           200                    // 最小输出缓冲区阈值
#define STORE_BUF_SIZE           236                    // 内部存储缓冲区大小
```
