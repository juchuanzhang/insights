# 微架构瞬态执行漏洞防护机制 · PPT提纲

> 基于《微架构瞬态执行漏洞防护机制洞察》v1.0（2026年5月）

---

## 一、首页

### 一句话总结

> **从Spectre到IPRED：2018年至今，CPU推测执行漏洞驱动了一场横跨硅片→微码→内核→编译器→虚拟化→浏览器的全栈防御体系重构，旧硬件性能损失可达60%，新硬件正从"打补丁"迈向"架构原生免疫"。**

### 洞察方向

现代高性能处理器通过推测执行（Speculative Execution）激进优化性能，但其在推测路径上的微架构副作用（缓存、分支预测器、填充缓冲器等）催生了数十种侧信道攻击变体，使得非特权攻击者可以窃取内核内存、跨进程数据、虚拟机机密甚至SGX飞地密钥。本洞察系统性梳理 Intel/AMD/ARM 硬件防御、Linux内核软件防御、编译器/虚拟化/浏览器防御的完整技术栈及其效能对比。

### 关键趋势

1. **从被动打补丁到主动硬件免疫**：Intel Granite Rapids（IPRED）、AMD Zen 4、ARM Cortex-X3（CSV2_3）已将推测隔离硬化到硅片中
2. **防御纵深持续下沉**：缓解从OS内核向编译器/运行时/JIT引擎/浏览器沙箱逐层扩展
3. **性能与安全的永恒博弈**：旧硬件全套防御损失30-60%性能；新硬件降至1-5%，差距持续拉大
4. **SMT（同时多线程）仍是阿喀琉斯之踵**：Core Scheduling缓解但未根除，安全管理需权衡
5. **RISC-V开放硬件带来新机遇**：可审计的安全验证成为差异化竞争力

### 价值识别

| 价值维度 | 量化/定性描述 |
|----------|--------------|
| **安全合规** | 云厂商（AWS/Azure/GCP）均要求已修补硬件；未缓解系统面临合规风险 |
| **性能成本** | 旧硬件性能损失30-60%；新硬件<5%；驱动硬件更新换代的经济决策 |
| **跨层协同** | 仅硬件缓解不足，需硅片+微码+内核+编译器+运行时全栈配合 |
| **威胁面评估** | 跨特权级攻击防护度高；跨VM/SGX攻击仍有中低风险缺口 |
| **竞争力差异** | ARM架构原语（CSV2/SB/BTI）设计前瞻性优于x86的打补丁模式 |

### 技术识别

```
全栈防御技术栈：
┌──────────────────────────────────────────────────┐
│  浏览器层   │ 站点隔离 | 定时器降级 | JIT索引掩码   │
│  运行时层   │ prctl()推测控制 | 常量时间算法       │
│  编译器层   │ Retpoline | SLH | LFENCE屏障         │
│  内核层     │ KPTI | IBRS/eIBRS | VERW | RSB填充   │
│  虚拟化层   │ L1D Flush | Core Scheduling | IBPB    │
│  微码/固件  │ IBPB | BHI_DIS_S | AGESA | Zenbleed   │
│  硅片层     │ IPRED | CSV2_3 | DDP | FSFP          │
└──────────────────────────────────────────────────┘
```

### 重要玩家

| 角色 | 组织 | 代表性贡献 |
|------|------|-----------|
| **漏洞发现** | Google Project Zero | Spectre/Meltdown首次披露 |
| **漏洞发现** | ETH Zurich (ComSec) | Retbleed, Zenbleed |
| **漏洞发现** | VUSec (VU Amsterdam) | LVI, MDS系列 |
| **硬件防御** | Intel | eIBRS, IPRED, BHI_DIS_S, VERW |
| **硬件防御** | AMD | LFENCE序列化, PSFD, Inception缓解 |
| **硬件防御** | ARM | CSV2系列原语, SB指令, BTI, PAC |
| **OS内核** | Linux内核社区 | KPTI, Retpoline, prctl推测控制API |
| **编译器** | GCC/LLVM社区 | -mspeculative-load-hardening, Retpoline |
| **浏览器** | Google Chrome | 严格站点隔离, V8 JIT缓解 |
| **云平台** | AWS/Azure/GCP | 全栈安全加固与主机替换 |
| **虚拟化** | VMware/KVM/Xen/Hyper-V | HyperClear, Core Scheduler, VM边界清除 |

---

## 二、洞察资料汇总（重要文献 Top 30）

| # | 发布件 | 类型 | 发布者 | 单位 | 时间 | 一句话总结 |
|---|--------|------|--------|------|------|-----------|
| 1 | Spectre Attacks: Exploiting Speculative Execution | 学术论文 | Kocher, Horn等 | Google Project Zero | 2019.05 | 首次系统性揭示条件/间接分支推测执行可导致跨特权级内存泄漏 |
| 2 | Meltdown: Reading Kernel Memory from User Space | 学术论文 | Lipp, Schwarz, Gruss等 | TU Graz | 2018.08 | 利用异常时推测读取绕过内核内存隔离，主要影响Intel |
| 3 | A Systematic Evaluation of Transient Execution Attacks and Defenses | 学术论文 | Canella, Van Bulck等 | TU Graz/imec-DistriNet | 2019.08 | 系统化分类瞬态执行攻击与防御，首次提出统一分析框架 |
| 4 | LVI: Hijacking Transient Execution through Load Value Injection | 学术论文 | Van Bulck等 | imec-DistriNet | 2020.05 | 将攻击者数据注入SGX飞地推测执行路径，逆转SGX安全假设 |
| 5 | RETBLEED: Arbitrary Speculative Code Execution with Return Instructions | 学术论文 | Wikner, Razavi | ETH Zurich | 2022.08 | 利用RET指令推测劫持实现任意代码推测执行，影响Intel/AMD |
| 6 | Zenbleed (CVE-2023-20593) | 安全披露 | Ormandy, T. | Google Project Zero | 2023.07 | 发现AMD Zen 2寄存器重命名竞争条件导致跨线程寄存器泄漏 |
| 7 | RIDL: Rogue In-Flight Data Load (MDS系列) | 学术论文 | Van Schaik等 | VUSec | 2019.05 | 揭示CPU内部填充缓冲区中的未提交数据可被推测采样窃取 |
| 8 | GDS/Downfall (CVE-2022-40982) | 安全披露 | Intel / Moghimi | Intel / UCSD | 2023.08 | AVX指令集收集操作可泄漏SIMD寄存器中的陈旧数据 |
| 9 | KAISER: Hiding the Kernel from User Space | 学术论文 | Gruss等 | TU Graz | 2017.06 | 提出内核页表隔离技术(KPTI前身)，成为Meltdown软件防线基础 |
| 10 | Retpoline: Software Construct for Preventing BTI | 技术方案 | Turner, P. | Google | 2018.01 | 使用RET指令替代间接跳转，防止Spectre v2分支目标注入 |
| 11 | Speculative Load Hardening | 编译器方案 | Carruth, C. | Google/LLVM | 2018.10 | 编译器级推测加载强化，在条件分支后插入数据依赖保护代码 |
| 12 | Analysis of Speculative Execution Side Channels | 白皮书 | Intel | Intel | 2018.05 | Intel官方发布IBRS/STIBP/IBPB三种推测控制MSR完整定义 |
| 13 | Enhanced IBRS Technical Documentation | 技术文档 | Intel | Intel | 2019+ | eIBRS实现单次MSR写入即可跨特权级隔离，含自动STIBP |
| 14 | Branch History Injection (BHI) | 安全公告 | Intel | Intel | 2022.03 | 分支历史注入可使eIBRS失效，需BHI_DIS_S或软件序列缓解 |
| 15 | Software Techniques for Managing Speculation (AMD) | 白皮书 | AMD | AMD | 2018+ | AMD官方推测管理指南，明确LFENCE序列化行为与IBPB差异 |
| 16 | AMD-SB-7005: Inception (CVE-2023-20569) | 安全公告 | AMD | AMD | 2023.08 | Zen 3/4返回预测训练中毒漏洞，需增强IBPB/RSB清除 |
| 17 | Cache Speculation Side-channels (ARM) | 白皮书 | ARM | ARM | 2024.03 | ARM推测执行漏洞体系完整说明，涵盖CSV2/CSV3及架构响应 |
| 18 | FEAT_CSV2: Branch Prediction Isolation | 架构规范 | ARM | ARM | 2018+ | ARMv8.2+硬件自动跨特权级清除分支预测器，无需软件干预 |
| 19 | Linux Kernel PTI/KPTI Implementation | 内核源码 | Linux社区 | Linux Foundation | 2018+ | arch/x86/entry/entry_64.S实现用户/内核页表隔离 |
| 20 | /sys/devices/system/cpu/vulnerabilities | 内核特性 | Linux社区 | Linux Foundation | 2018+ | 通过sysfs实时暴露15+漏洞状态与缓解信息 |
| 21 | prctl() Speculation Control API | 内核API | Linux社区 | Linux Foundation | 2018+ | 用户态进程通过prctl自主控制SSBD和间接分支推测行为 |
| 22 | KVM Speculation Control & CPU Buffer Clearing | 虚拟化 | KVM社区 | Linux Foundation | 2019+ | VMEntry/VMExit时VERW清除与RSB填充，防御跨VM攻击 |
| 23 | Core Scheduling for SMT Protection | 内核特性 | Linux社区 | Linux Foundation | 2020+ | 确保互不信任的进程/VM不同时运行在SMT兄弟线程上 |
| 24 | Site Isolation (Chrome) | 浏览器方案 | Chromium Team | Google | 2018+ | 每个渲染进程限制于同一站点，消除Spectre跨站数据窃取 |
| 25 | Project Fission (Firefox) | 浏览器方案 | Mozilla | Mozilla | 2021 | Firefox站点隔离架构，每源独立渲染进程 |
| 26 | Spectre Mitigations in V8 JIT | JIT方案 | V8 Team | Google | 2018+ | 索引掩码+推测污染追踪+LFENCE插入保护JS引擎 |
| 27 | BoringSSL Side Channel Defenses | 密码学库 | Google | Google | 持续更新 | 严格常量时间算法+推测屏障原语保护密钥操作 |
| 28 | JDK 11 Speculative Execution Mitigations | 运行时 | Oracle | Oracle | 2018 | Java JIT使用索引掩码+AOT编译器支持Retpoline |
| 29 | Intel IPRED & DDP Hardware Protections | 硬件特性 | Intel | Intel | 2024 | Granite Rapids引入间接预测器隔离与数据依赖预取器 |
| 30 | Hardware-Software Contracts for Secure Speculation | 学术论文 | Guarnieri等 | UC Riverside | 2021.05 | 提出硬件-软件安全推测合约的形式化框架 |

---

## 三、PPT正文

### 【Slide N: WHY】—— 为什么这个洞察重要？

**核心问题**：现代CPU的性能基石——推测执行，本身就是一个系统级安全漏洞的根源。

- **2018年1月**，Google Project Zero公开披露Spectre和Meltdown，标志着高性能计算进入"推测安全"时代
- 此后**19种以上**瞬态执行攻击变体被发现，覆盖Intel/AMD/ARM所有主流架构
- 攻击效果：非特权用户态代码 → 窃取内核内存 → 跨VM窃取宿主数据 → 窃取SGX飞地密钥 → 浏览器内窃取跨站数据
- **根本矛盾**：推测执行提升性能的核心机制（分支预测、缓存预取、乱序执行）与安全隔离原则（特权级、进程、VM、站点边界）存在结构性冲突

**为什么到今天仍重要**（2026年视角）：
- 仍有数十亿旧设备运行未修复或部分修复的系统
- 新攻击面不断被发现（ITS/IBPD, 2024年）
- 防御措施的累积性能开销迫使企业在安全与成本间艰难权衡

> **一句话：这不是一个被解决的历史问题，而是正在重塑CPU架构设计、操作系统内核、语言运行时的持续演进过程。**

---

### 【Slide N+1: WHAT-1】—— 攻击本质：瞬态执行与微架构侧信道

**核心概念**：现代CPU为提升性能，在分支/数据依赖未确认前即"猜测"执行后续指令。当猜测正确，性能大幅提升；当猜测错误，架构状态回滚——但微架构状态的改变不会回滚。

```
正常执行路径：
  取指 → 译码 → 执行 → 访存 → 写回 → 提交 (Architectural State)

推测执行路径：
  取指 → 译码 → 推测执行 → 预测错误 → 丢弃架构结果
                            ↓
                    微架构痕迹保留 (缓存/预测器/缓冲区) ← 攻击者观测点
```

**什么是微架构侧信道？**

| 信道类型 | 利用原理 | 精度 | 代表攻击 |
|----------|---------|------|----------|
| **缓存计时 (Cache Timing)** | 测量内存访问延迟推断缓存命中/未命中 | 64B 缓存行粒度 | Spectre v1/v2, Meltdown |
| **分支预测器状** | 观测分支预测器训练状态推断历史分支方向 | 位级 | BHI, Spectre v2 |
| **TLB 状态** | 推断页表遍历缓存状况 | 4KB 页面粒度 | TLBleed |
| **端口争用** | CPU执行端口竞争导致可测延迟 | 指令级 | SMoTherSpectre |
| **AVX/SSE 寄存器有效期** | SIMD寄存器在上下文切换间的残留 | 128-512位 | GDS/Downfall, Zenbleed |

**为什么这不同于传统漏洞？**
- 传统漏洞：利用软件Bug（溢出、UAF）→ 可修代码
- 瞬态执行漏洞：利用**硬件设计特性** → 修复需改动硅片，软件缓解是"止血"而非"治愈"

---

### 【Slide N+2: WHAT-2】—— 攻击分类体系：19种攻击全景图

**分类维度一：按攻击机制**

| 类别 | 攻击原理 | 代表 CVE | 影响范围 | 严重性 |
|------|----------|----------|----------|--------|
| **Spectre v1** | 条件分支推测绕过边界检查 | CVE-2017-5753 | Intel/AMD/ARM 全系 | ★★★★ |
| **Spectre v2** | 间接分支目标注入(BTI) | CVE-2017-5715 | Intel/AMD/ARM 全系 | ★★★★ |
| **Meltdown** | 异常时推测读取内核内存 | CVE-2017-5754 | 主要 Intel，部分 ARM | ★★★★★ |
| **Spectre v3a** | 系统寄存器推测读取 | CVE-2018-3640 | Intel/ARM | ★★★ |
| **Spectre v4** | 推测性存储绕过(SSB) | CVE-2018-3639 | Intel/AMD/ARM 全系 | ★★★ |
| **L1TF** | L1终端故障 | CVE-2018-3615 | Intel | ★★★★★ |
| **MDS** | 微架构数据采样(4子变体) | CVE-2018-12126系列 | Intel | ★★★★ |
| **TAA** | TSX异步中止 | CVE-2019-11135 | Intel(TSX处理器) | ★★★ |
| **LVI** | 加载值注入 | CVE-2020-0551 | Intel(SGX场景) | ★★★★ |
| **SRBDS** | 特殊寄存器数据采样 | CVE-2020-0543 | Intel | ★★★ |
| **BHI** | 分支历史注入 | CVE-2022-0001 | Intel/ARM | ★★★★ |
| **Retbleed** | 返回指令推测劫持 | CVE-2022-29900 | Intel/AMD | ★★★★★ |
| **GDS** | 收集数据采样 | CVE-2022-40982 | Intel(AVX) | ★★★★ |
| **Zenbleed** | AMD Zen 2寄存器泄漏 | CVE-2023-20593 | AMD Zen 2 | ★★★★ |
| **Inception** | AMD返回预测训练中毒 | CVE-2023-20569 | AMD Zen 3/4 | ★★★★ |
| **RFDS** | 寄存器文件数据采样 | CVE-2023-28746 | Intel Atom | ★★★ |
| **Native BHI** | 原生分支历史注入 | CVE-2024-2201 | Intel | ★★★★ |
| **ITS** | 间接目标选择 | CVE-2024-28956 | Intel | ★★★ |
| **IBPD** | 间接分支预测延迟更新 | CVE-2024-45332 | Intel | ★★★ |

**分类维度二：按攻击目标（受影响资产）**

```
跨特权级攻击 (用户→内核)
  Meltdown, L1TF, MDS, Spectre v3a, SWAPGS
         ↓
跨进程攻击 (进程A→进程B)
  Spectre v2, BHI, Retbleed, Spectre v4
         ↓
跨VM攻击 (Guest→Host / Guest→Guest)
  L1TF, Spectre v2, MDS (SMT同核)
         ↓
SGX飞地攻击
  LVI, SGAxe, Foreshadow
         ↓
跨站攻击 (浏览器内)
  Spectre v1 (JavaScript JIT gadget)
```

---

### 【Slide N+3: WHAT-3】—— 三大根本原因深度剖析

**根因一：推测执行绕过权限检查**

```
正常流程：          推测攻击流程：
检查权限 ────→ 允许访问内存   检查权限(已发起) ──→ 推测读取 ──→ 权限检查未完成
   │                               ↓                        ↓
   └──→ 若非法则拒绝          数据已进入缓存        权限检查返回"拒绝"
                                 ↓                        ↓
                           架构结果丢弃          ⚠ 缓存痕迹保留
                                                    ↓
                                         攻击者通过Flush+Reload测时间
                                         → 推断被访问的缓存行 → 泄露数据
```

受影响的检查类型：页表权限位(U/S)、段寄存器、xAPIC访问控制、SGX飞地边界

**根因二：分支预测器污染**

CPU内部预测器结构及被利用方式：

| 预测器结构 | 功能 | 攻击方式 | 污染手段 |
|-----------|------|---------|---------|
| **BTB** (Branch Target Buffer) | 缓存间接分支目标地址 | Spectre v2 | 在攻击者地址空间中训练，使其跳转到受害者gadget |
| **BHB** (Branch History Buffer) | 记录最近N次分支方向 | BHI | 用精心编排的分支序列训练历史，使eIBRS被绕过 |
| **RSB** (Return Stack Buffer) | 缓存CALL/RET配对地址 | Retbleed | 使RSB下溢回退到BTB预测，劫持返回地址 |
| **RAS** (Return Address Stack) | AMD返回地址预测 | Inception | 训练返回预测器使其在推测路径跳至攻击者地址 |

**根因三：CPU内部缓冲区陈旧数据采样**

```
CPU核心内部：
┌────────────────────────────────────────────┐
│  架构寄存器 (AX/BX/...) ← 正常可见         │
│  重命名寄存器 (192+ entries)                │
│  存储缓冲区 (Store Buffer, 56 entries)      │ ← MDS
│  加载缓冲区 (Load Buffer, 72 entries)       │ ← MDS
│  填充缓冲区 (Fill Buffer, 12 entries)       │ ← MDS, TAA, RFDS
│  行填充缓冲区 (Line Fill Buffer)            │ ← L1TF, SRBDS
│  SIMD寄存器文件 (AVX/SSE)                  │ ← GDS
└────────────────────────────────────────────┘

攻击模式：攻击者触发受害者数据填充缓冲区 → 受害者数据残留在缓冲区
→ 攻击者通过推测执行读取残留数据 → 数据进入缓存
→ 攻击者通过Flush+Reload侧信道观测 → 数据泄露
```

---

### 【Slide N+4: WHAT-4】—— Intel防御技术演进：从补丁到免疫

```
2018 · Legacy IBRS
  ├── 每次内核进入/退出均需写MSR → 高开销
  ├── 需配合Retpoline使用
  └── 仅Skylake及更早平台
        ↓
2019 · Enhanced IBRS (eIBRS) ── Cascade Lake+
  ├── 一次设置，硬件自动在用户态返回时隔离BTB
  ├── 自动包含STIBP（跨线程保护）
  ├── 性能开销大幅降低
  └── ⚠ 不能防御BHI攻击
        ↓
2022 · BHI_DIS_S + eIBRS ── Alder Lake+
  ├── 禁用分支历史缓冲区对间址预测的影响
  ├── 仅支持eIBRS且BHI_NO=0的处理器需要
  └── 性能影响：0-15%
        ↓
2024 · IPRED (Indirect Predictor) ── Granite Rapids+
  ├── 间接预测器与直接预测器硬件隔离
  ├── 模式标签(Modality Tags)防跨特权级污染
  ├── 同时缓解ITS和IBPD
  └── 性能影响：可忽略（硬件原生设计）
```

**Intel关键MSR寄存器一览：**

| MSR | 地址 | 核心位 | 功能 |
|-----|------|--------|------|
| IA32_SPEC_CTRL | 0x48 | Bit0(IBRS), Bit1(STIBP), Bit2(SSBD) | 推测执行行为控制 |
| IA32_PRED_CMD | 0x49 | Bit0(IBPB) | 触发分支预测器刷新 |
| IA32_ARCH_CAPABILITIES | 0x10A | Bit0~25（多项能力位） | 只读安全能力报告 |

**Intel不同微架构世代的防御能力演变：**

```
Sandy Bridge (2011) ──────── 全部受影响，仅Legacy IBRS可用
Ivy Bridge   ────────  ↑
Haswell (2013) ──────── MDS/TAA开始影响
Broadwell    ────────  ↑
Skylake (2015) ──────── 防御全面依赖微码+软件组合
Kaby Lake    ────────  ↑
Coffee Lake (2018) ──── Meltdown硬件修复
Cascade Lake (2019) ─── eIBRS + MDS/L1TF/TAA硬件修复
Ice Lake     ────────  ↑
Tiger Lake   ────────  ↑
Alder Lake (2021) ──── BHI_DIS_S加入
Raptor Lake (2022) ─── MDS/GDS硬件修复
Sapphire Rapids (2023) ─── BHI_DIS_S全覆盖
Granite Rapids (2024) ─── IPRED + DDP → 新生代硬件免疫
```

---

### 【Slide N+5: WHAT-5】—— AMD/ARM防御体系对比

**AMD：不同于Intel的推测控制哲学**

| 对比维度 | Intel | AMD | 影响 |
|----------|-------|-----|------|
| LFENCE行为 | 非序列化（旧架构） | **始终序列化** → 天然Spectre v1屏障 | AMD对Spectre v1防御更强 |
| IBPB行为 | 刷新BTB+RSB | 刷新BTB，**不刷新RSB**（需显式填充） ⚠ | AMD需额外处理Retbleed/Inception |
| Meltdown | 受影响（旧架构） | **架构级不受影响**（特权级预测检查） | AMD天然优势 |
| MDS/GDS/L1TF | 广泛受影响 | **完全不受影响** | AMD缓冲区设计差异性保护 |
| 固件机制 | 微码 + ucode包 | AGESA BIOS固件更新 | AMD更依赖OEM/主板商推送 |

**AMD关键安全事件：**
- **Zenbleed (2023)**：Zen 2独有——VZEROUPPER优化 + 寄存器重命名竞争 → 跨线程寄存器泄漏 → AGESA ComboAM4v2 PI 1.2.0.A修复
- **Inception (2023)**：Zen 3/4——返回预测器中毒 → 增强IBPB/RSB刷新 → AGESA ComboAM4v2 PI 1.2.0.B修复
- **PSFD**：Zen 3+特性——Predictive Store Forwarding Disable (MSR 0x48 Bit 3)，防止推测存储转发侧信道

**ARM：架构前向设计带来原生成熟度**

| 架构级别 | 关键特性 | 时间 | 防御效果 |
|----------|---------|------|---------|
| ARMv8.0 | SB指令 + SSBD | 2011 | Spectre v1/v4基本防御 |
| ARMv8.2+ | CSV2_1p1 | 2016 | 硬件自动EL0↔EL1分支隔离，无需内核干预 |
| ARMv8.2+ | CSV2_1p2 | 2017 | 全特权级(EL0↔EL1↔EL2)硬件隔离 |
| ARMv8.4+ | CSV2_2 | 2019 | 增强版预测器隔离 |
| ARMv8.5+ | CSV2_3 + BTI强制 | 2020 | 最全面硬件隔离 + 分支目标识别 |
| ARMv8.3-A | PAC (指针认证) | 2016 | 密码学签名防ROP，间接防御Spectre v2利用链 |

**ARM vs x86关键差异：ARM的几乎所有Meltdown/L1TF/MDS/GDS类攻击天然不受影响**

---

### 【Slide N+6: WHAT-6】—— 全栈防御分层模型

**为什么单层防御永远不够？——以Spectre v2为例的多层防御链**

```
Spectre v2 攻击面 (间接分支推测)
    │
    ├── 硬件层：eIBRS/CSV2 → 隔离跨特权级BTB
    │   └── 失效场景：BHI绕过eIBRS → 需BHI_DIS_S/IPRED
    │
    ├── 微码层：IBPB → 上下文切换时刷新预测器
    │   └── 失效场景：Retbleed(RSB下溢)绕过IBPB → 需RSB填充
    │
    ├── 内核层：Retpoline + RSB填充 → 替代间接跳转 + 防RSB下溢
    │   └── 失效场景：SMT跨线程攻击 → 需STIBP + Core Scheduling
    │
    ├── 编译器层：Retpoline编译选项 → 代码生成级防御
    │   └── 失效场景：手写汇编/旧二进制 → 需IBC(Indirect Branch Control)
    │
    └── 用户态层：prctl(PR_SPEC_INDIRECT_BRANCH) → 进程级牺牲性能换安全
```

**防御分层全景（附原理与防护目标描述）：**

```
┌──────────────────────────────────────────────────────────────────────┐
│ 第七层 · 用户应用                                                     │
├──────────────────────────────────────────────────────────────────────┤
│ 常量时间算法      │ 内存访问模式与密钥值无关 → 防 Spectre v1/v2 缓存时差│
│ prctl()推测控制   │ 用户进程系统调用自主开关IBPB/SSBD → 防 Spectre v2/v4│
│ 内存清除          │ OPENSSL_cleanse()显式擦除残留 → 防 MDS/TAA/L1TF采样 │
├──────────────────────────────────────────────────────────────────────┤
│ 第六层 · 浏览器/JS运行时                                              │
├──────────────────────────────────────────────────────────────────────┤
│ 站点隔离          │ 每站点独立渲染进程 → 防 Spectre v1/v2 跨站数据窃取  │
│ 定时器降级        │ performance.now()精度5μs→100μs → 防所有Flush+Reload │
│ JIT索引掩码       │ 数组边界检查+CMOV掩码 → 防 Spectre v1 JS引擎内推测  │
│ 推测污染追踪      │ 推测路径值禁止用于后续地址计算 → 防 Spectre v1 利用链 │
├──────────────────────────────────────────────────────────────────────┤
│ 第五层 · 编译器工具链                                                  │
├──────────────────────────────────────────────────────────────────────┤
│ Retpoline         │ RET替代JMP*reg，间址调用转RSB预测 → 防 Spectre v2   │
│ SLH推测加载强化   │ cmp后插CMOV+掩码构造数据依赖 → 防 Spectre v1 分支绕过│
│ LFENCE/CSDB屏障   │ 阻止后续指令在条件解析前推测执行 → 防 Spectre v1     │
│ LVI硬化           │ 每条可被利用的load后插LFENCE → 防 LVI (加载值注入)   │
├──────────────────────────────────────────────────────────────────────┤
│ 第四层 · 虚拟化平台                                                    │
├──────────────────────────────────────────────────────────────────────┤
│ L1D Flush         │ VMEntry前wrmsr清除L1D中的陈旧PTE → 防 L1TF 跨VM    │
│ VERW (VM)         │ VMEntry时VERW清除CPU缓冲区 → 防 MDS/TAA/RFDS 跨VM  │
│ RSB填充 (VM)      │ VMExit时call 32次防RSB下溢 → 防 Retbleed 跨VM      │
│ IBPB (VM)         │ VM切换时写MSR触发预测器刷新 → 防 Spectre v2 跨VM   │
│ Core Scheduling   │ 互不信任VM不同时在SMT兄弟线程上 → 防 MDS/L1TF/Spectre│
├──────────────────────────────────────────────────────────────────────┤
│ 第三层 · 操作系统内核                                                  │
├──────────────────────────────────────────────────────────────────────┤
│ KPTI              │ 用户态内核页表近乎为空 → 防 Meltdown 跨特权级       │
│ Retpoline (内核)  │ 内核中间接调用全部走retpoline thunk → 防 Spectre v2 │
│ IBRS/eIBRS        │ 写MSR让CPU自动跨特权级隔离BTB → 防 Spectre v2/Retbleed│
│ IBPB (调度)       │ 上下文切换到敏感进程时写MSR刷新 → 防 Spectre v2 跨进程│
│ STIBP             │ 写MSR隔离SMT兄弟线程间址预测 → 防 Spectre v2 跨SMT  │
│ VERW (内核)       │ 退出内核态时执行VERW清除缓冲区 → 防 MDS/TAA/RFDS/SRBDS│
│ RSB填充 (调度)    │ 上下文切换时call 32次填充RSB → 防 Retbleed/Spectre v2│
│ Core Scheduling   │ 互不信任进程不同时在SMT兄弟线程上 → 防 MDS/MLPDS 跨线程│
│ SMT控制           │ 通过/sys/.../smt/control禁用SMT → 防所有SMT共享通道攻击│
│ /sys/vulnerabilities│ 实时暴露15+漏洞状态 → 不直接防护，提供安全态势可见性│
├──────────────────────────────────────────────────────────────────────┤
│ 第二层 · 微码/固件                                                     │
├──────────────────────────────────────────────────────────────────────┤
│ IBRS              │ MSR 0x48 Bit0: 限制间接分支推测跨特权级 → 防 Spectre v2│
│ STIBP             │ MSR 0x48 Bit1: SMT下隔离间址预测器 → 防 Spectre v2 跨线程│
│ SSBD              │ MSR 0x48 Bit2: 禁止推测性存储绕过 → 防 Spectre v4  │
│ IBPB              │ MSR 0x49 Bit0: 触发预测器硬件刷新 → 防 Spectre v2 跨进程│
│ BHI_DIS_S         │ 微码控制位禁用BHB对间址预测影响 → 防 BHI (eIBRS绕过)│
│ VERW清除          │ 微码劫持VERW指令清除Fill/Load/Store Buffer → 防 MDS系列│
│ AGESA修复         │ AMD BIOS固件: DE_CFG[9]等寄存器修复 → 防 Zenbleed/Inception│
│ TF-A              │ ARM固件提供推测控制参考实现 → 防 Spectre v2/BHI (ARM)│
├──────────────────────────────────────────────────────────────────────┤
│ 第一层 · 硅片硬件                                                      │
├──────────────────────────────────────────────────────────────────────┤
│ IPRED             │ 间/直接预测器物理隔离+模式标签 → 防 Spectre v2/BHI/ITS/IBPD│
│ CSV2_1p1/1p2/2/3  │ ARM硬件特权切换时自动清分支预测器 → 防 Spectre v2 (ARM)│
│ DDP               │ 预取仅基于数据流依赖非控制流 → 防 数据依赖预取器侧信道│
│ FSFP              │ 写转发预测仅基于已完成的store-load对 → 防存储转发侧信道│
│ LFENCE序列化      │ AMD LFENCE在推测路径也阻止后续发射 → 防 Spectre v1  │
│ BTI/PAC           │ 间址目标合法标记+指针密码签名 → 防 Spectre v2 ROP/JOP链│
│ 硬件Meltdown免疫  │ 推测执行中完成权限验证后才访存 → 防 Meltdown (CVE-5754)│
└──────────────────────────────────────────────────────────────────────┘
```

**关键认知**：任何单层被绕过 ≠ 全线崩溃，但不覆盖所有层 = 攻击面持续存在

---

### 【Slide N+7: WHAT-7】—— 防御效能量化：性能影响矩阵

**核心发现：新硬件已将防御性能损失从60%压缩至5%以内**

| 防御机制 | Intel Skylake (旧) | Intel Cascade Lake | Intel Granite Rapids | AMD Zen 3 | ARM N1 |
|----------|-------------------|--------------------|---------------------|-----------|--------|
| **KPTI** | -15~30% | -2~5% (PCID+HW修复) | <1% | 0% (不受影响) | 0~3% |
| **Spectre v2** | -5~10% (Retpoline) | 0% (eIBRS) | 0% (IPRED) | 0% (Auto IBRS) | 0% (CSV2_1p1) |
| **MDS/VERW** | -3~8% | -0.5~2% | 0% (HW修复) | 0% | 0% |
| **L1TF** | -5~15% (含SMT) | -0.5~3% | 0% | 0% | 0% |
| **BHI** | -2~10% | 0% (BHI_DIS_S) | 0% (IPRED) | -0.5~2% | 0% (CSV2_1p2) |
| **Retbleed** | -2~14% (RSB) | -0.5~2% | 0% | -2~8% (Zen1/2/3) | -0.5~2% |
| **GDS** | N/A | -5~10% | 0% | 0% | 0% |
| **全部启用(最坏)** | **-30~60%** | **-5~15%** | **-2~5%** | **-2~10%** | **-1~5%** |

**按威胁场景的防护成熟度评估：**

| 威胁场景 | 防护等级 | 剩余风险 |
|----------|---------|---------|
| 用户态→内核态 | ★★★★ 高 | 旧款Intel需全栈启用，性能代价大 |
| Guest→Host | ★★★★ 高 | Core Scheduling未启用即存在SMT通道 |
| Guest→Guest | ★★★ 中 | IBPB频率不足时仍可泄露 |
| SGX飞地 | ★★★ 中 | LVI缓解完整但仅限SGX专用编译 |
| JS→跨站数据 | ★★★★ 高 | 站点隔离覆盖面有内存开销 |
| IoT/嵌入式 | ★ 低 | 老旧ARM设备基本无防御手段 |
| 5G/边缘加速器 | ★★ 低 | GPU/NPU推测行为研究不充分 |

---

### 【Slide N+8: WHAT-8】—— 攻防演进脉络：为什么防御反复被绕过？

> 核心视角：每个新攻击都不是"发现新的微架构结构"，而是**发现已有防御的覆盖盲区**。

---

**绕过路线图总览：**

```
防御链                                                       绕过突破口
──────────────────────────────────────────────────────────────────────

KPTI (页表隔离) ───────────── 被认为终结 Meltdown ──→ ECDSA密钥仍残留
  │                                                    L1缓存（L1TF, 2018.08）
  │                                                    页表项反转再被绕
  │                                                    （MDS采样, 2019.05）
  │
IBRS + Retpoline ──────────── 被认为终结 Spectre v2 ──→ IBRS只隔离BTB，
  │                                                    不隔离BHB（BHI, 2022.03）
  │                                                    Retpoline只替换间接跳转，
  │                                                    不替换RET（Retbleed, 2022.07）
  │
VERW 缓冲清除 ─────────────── 被认为终结 MDS ─────────→ 新发现的缓冲区类型
  │                                                    （TAA: TSX缓冲, 2019.11）
  │                                                    （SRBDS: 特殊寄存器, 2020.06）
  │                                                    （RFDS: Atom寄存器文件, 2023）
  │
eIBRS + BHI_DIS_S ─────────── 被认为终结 BHI ─────────→ 仍依赖软件序列，
  │                                                    原生BHI无需eBPF即可利用
  │                                                    (Native BHI, 2024.03)
  │                                                    间接预测器更新时机窗口
  │                                                    (ITS + IBPD, 2024.10-12)
```

---

**关键绕过事件深度分析：**

### 绕过1：L1TF — KPTI的"盲区"

```
KPTI 如何工作的？                     L1TF 如何绕过的？
┌──────────────────────┐             ┌──────────────────────┐
│ 用户态运行时：        │             │ CPU推测执行时：       │
│ 内核页表几乎为空      │             │ L1缓存中仍有内核PTE   │
│ 访问内核地址 → 页错误 │             │ 推测路径在页错误前    │
│ → 阻止Meltdown        │             │ 读取了L1缓存中的PTE   │
└──────────────────────┘             │ → 推测访问了内核物理页 │
        ✓ 完美防御                  │ → 数据进入L1缓存       │
                                     └──────────────────────┘
                                              ↓
                                      ⚠ KPTI针对页表，L1TF针对
                                        L1缓存中的陈旧PTE副本
```

**后续演化**：L1TF的PTE反转缓解 → 又被MDS绕过（不需PTE，直接采样CPU内部缓冲区中的残留数据）

### 绕过2：BHI — eIBRS的"盲区"

```
eIBRS 如何工作的？                   BHI 如何绕过的？
┌──────────────────────┐             ┌──────────────────────┐
│ IBRS=1时：            │             │ eIBRS只隔离BTB：      │
│ 硬件在特权级切换时     │             │ 分支目标地址缓存的    │
│ 隔离分支目标缓冲区(BTB)│             │ 跨特权级查找被阻止    │
│ 用户态无法注入内核BTB │             │ ✓ 传统Spectre v2无效  │
└──────────────────────┘             └──────────────────────┘
        ✓ BTB隔离                            ↓
                                     ⚠ 但分支历史缓冲区(BHB)未被隔离！
                                     ┌──────────────────────┐
                                     │ BHB记录最近N次分支方向│
                                     │ 攻击者在用户态精心编排│
                                     │ 分支序列训练BHB       │
                                     │ → 内核进入后BHB状态   │
                                     │   仍指向攻击者地址    │
                                     │ → 间接分支推测跳转    │
                                     │   到攻击者gadget      │
                                     └──────────────────────┘
```

### 绕过3：Retbleed — Retpoline的"盲区"

```
Retpoline 如何工作的？                Retbleed 如何绕过的？
┌──────────────────────┐             ┌──────────────────────┐
│ 用 RET 替代 JMP *reg  │             │ Retpoline替换了间址调用│
│ RET使用RSB预测而非BTB │             │ 但没有替换函数返回！  │
│ 推测执行时在无限循环  │             │ 正常的 RET 指令仍然   │
│ 中等待，防止跳转gadget│             │ 存在，且也使用RSB预测 │
└──────────────────────┘             └──────────────────────┘
        ✓ 间接调用被保护                        ↓
                                     ⚠ RSB深度有限（16-32 entries）
                                     ┌──────────────────────┐
                                     │ 攻击者通过深层调用栈  │
                                     │ 使RSB下溢（耗尽）     │
                                     │ → RSB下溢时回退到BTB  │
                                     │   或替代预测器        │
                                     │ → 函数返回被劫持      │
                                     │ → 这绕过了Retpoline！ │
                                     └──────────────────────┘
```

### 绕过4：AMD IBPB的"盲区" → Inception

```
Intel IBPB：                          AMD IBPB：
刷新BTB + RSB                       只刷新BTB，不刷新RSB！
     ✓                                    ⚠
                                     攻击者训练返回预测器
                                     → IBPB执行后RSB状态仍被污染
                                     → 此后RET指令推测跳转至攻击地址
                                     → Inception (2023)
                                     修复：增强IBPB同时刷RSB
```

---

**攻防演进的核心模式：**

```
每一轮攻防都遵循相同的"覆盖盲区"模式：

   防御发布 ──→ 研究者分析防御覆盖了什么？没覆盖什么？
                    ↓
              找到未覆盖的微架构状态 × 可推测访问的时机窗口
                    ↓
              设计PoC：训练微架构状态 → 触发推测窗口 → 侧信道观测
                    ↓
              新攻击发布 ──→ 更强防御扩展现有机制的覆盖范围
                              (微码/MSR新增位/软件序列)
                    ↓
              覆盖范围再被分析... → 循环往复
                    ↓
              终极路径：硅片级硬件重设计，从结构上消除覆盖盲区
              (IPRED: 间接/直接预测器物理隔离, 无法跨类型污染)
              (DDP: 预取决策仅基于数据流, 攻击者无法控制)
              (CSV2_3: 所有特权级切换时硬件自动清全部预测状态)
```



---

### 【Slide N+9: WHAT-9】—— 攻击-vs-防御覆盖矩阵

> 表格含义：● 完全覆盖 | ◐ 部分覆盖(需特定配置/微码版本) | ○ 无覆盖/不受影响
>
> 防御层级缩写：**[硅]** 硅片 | **[微]** 微码/固件 | **[核]** 内核 | **[编]** 编译器 | **[虚]** 虚拟化 | **[应]** 用户态API

**防御覆盖总览矩阵（按从硅片到应用的层级排列）：**

| 攻击/CVE | IPRED<br>[硅] | IBRS<br>[微] | IBPB<br>[微] | STIBP<br>[微] | SSBD<br>[微] | VERW<br>[微] | BHI_<br>DIS_S<br>[微] | AGESA<br>[固件] | KPTI<br>[核] | RSB<br>填充<br>[核] | Core<br>Sched<br>[核] | L1D<br>Flush<br>[虚] | Retpoline<br>[编] | LFENCE<br>[编] | SLH<br>[编] |
|----------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Meltdown** | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ |
| **Spectre v1** | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ◐ |
| **Spectre 1.1** (2018-3693) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ◐ | ◐ |
| **Spectre v2** | ● | ● | ◐ | ◐ | ○ | ○ | ○ | ○ | ○ | ○ | ◐ | ○ | ● | ○ | ○ |
| **Spectre v4** | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **L1TF** | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ◐ | ● | ○ | ○ | ○ |
| **MDS** | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ◐ | ○ | ○ | ○ | ○ |
| **TAA** | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ◐ | ○ | ○ | ○ | ○ |
| **LVI** | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ |
| **SRBDS** | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **CVE-2021-0086** (FP) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **CVE-2021-0089** (FP) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **BHI** | ● | ◐ | ○ | ◐ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ◐ | ○ | ○ |
| **Retbleed** | ● | ◐ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ◐ | ○ | ○ |
| **CVE-2024-2193** (GhostRace) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ◐ | ○ |
| **GDS** | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **Zenbleed** | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **Inception** | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **RFDS** | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **Native BHI** | ● | ◐ | ○ | ◐ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ◐ | ○ | ○ |
| **ITS** | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **IBPD** | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **AMD CVE-2024-36348** (UMIP绕过) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **AMD CVE-2024-36349** (TSC_AUX) | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **AMD CVE-2024-36350** (存储推断) | ○ | ○ | ○ | ○ | ◐ | ○ | ○ | ● | ○ | ○ | ◐ | ◐ | ○ | ○ | ○ |
| **AMD CVE-2024-36357** (L1D缓存) | ○ | ○ | ◐ | ○ | ○ | ○ | ○ | ● | ◐ | ○ | ◐ | ◐ | ○ | ○ | ○ |

**各层级防御密度统计：**

| 防御层级 | 手段数量 | 覆盖攻击数 | 特点 |
|----------|---------|-----------|------|
| **[硅]** 硅片硬件 | 1 | 5 (3●直接 + 2●级联) | 一次设计永久免疫，但换代周期最长 |
| **[微]** 微码 | 6 | 覆盖18/26种 | 覆盖面最广、响应最快，旧硬件终止则停更 |
| **[固件]** BIOS/UEFI | 1 | 5 (Zenbleed + AMD-SB-7029×4) | AMD AGESA是AMD特有漏洞的唯一修复路径 |
| **[核]** 操作系统内核 | 3 | 补位SMT/RSB等死角 | 覆盖广但性能代价最大 |
| **[虚]** 虚拟化 | 1 | 部分覆盖L1TF + AMD存储/L1D | 仅保护VM边界，VM内部不负责 |
| **[编]** 编译器 | 3 | Spectre v1/v1.1/v2/LVI/GhostRace | 需源码重编译，对闭源/旧二进制无效 |
| **[应]** 用户态API | 0 (prctl辅助) | 定制化防护 | 粒度最细，但需进程主动调用 |

**新增漏洞未覆盖原因说明：**

| CVE | 名前 | 主要防御 | 未覆盖(○)的原因 |
|-----|------|---------|----------------|
| CVE-2018-3693 | Spectre 1.1 (BCBS) | LFENCE ◐ + SLH ◐ | 与Spectre v1同类（推测边界检查绕过），但涉及推测性存储。现有LFENCE/SLH屏障同样有效；其他防御手段针对不同攻击类别 |
| CVE-2021-0086 | Intel FP差异泄漏 | Intel微码SA-00516 | **非瞬态执行漏洞**，属CWE-203可观测差异。利用浮点指令响应时间差泄密，不在推测执行防御覆盖范围 |
| CVE-2021-0089 | Intel可观测差异 | Intel微码SA-00516 | 同上，非瞬态执行漏洞 |
| CVE-2024-2193 | GhostRace (SRC) | LFENCE ◐ | 推测性竞态条件。缓解通过在同步原语中插入LFENCE→内核补丁方式，SLH不覆盖内核代码 |
| CVE-2024-36348/49/50/57 | AMD Transient Scheduler | AMD AGESA SB-7029 | 2025年7月新披露的AMD瞬态调度器攻击族。AGESA固件是唯一通用修复路径。部分漏洞(36350/36357)可通过SSBD/Core Sched/L1D Flush/KPTI获得部分缓解 |

**按攻击类别的推荐防御组合：**

| 攻击类别 | 最小推荐组合 | 最佳推荐组合 | 未覆盖的残余风险 |
|----------|-------------|-------------|----------------|
| 推测绕过权限检查 | KPTI + 编译器SLH | KPTI + SLH + 硬件Meltdown免疫 | 旧硬件KPTI性能代价大 |
| 分支预测器污染 | Retpoline + IBPB | eIBRS + BHI_DIS_S + IPRED | eIBRS被BHI绕过（旧硬件无BHI_DIS_S）|
| 缓冲区陈旧数据采样 | VERW (退出内核时) | VERW + SMT禁用/ Core Scheduling | SMT同核线程仍可攻击（如未禁用SMT）|
| 存储推测绕过 | SSBD (MSR控制) | SSBD + prctl()按进程控制 | 默认off模式下非seccomp进程未保护 |
| 返回地址预测劫持 | RSB填充 + IBPB | eIBRS + RSB填充 + IPRED | AMD Zen 1/2/3 RSB填充性能影响较大 |
| 推测性竞态条件 (GhostRace) | 内核同步原语加LFENCE | 硬件推测屏障指令(SB)全覆盖 | 需识别所有敏感竞争点，易遗漏 |
| AMD瞬态调度器族 (SB-7029) | 等待AGESA更新 | AGESA + SMT禁用 + L1D Flush(VM) | AGESA推送依赖OEM；部分缓解仅限VM边界 |
| 非推测执行侧信道 (FP差异) | Intel微码更新 | 微码 + 应用层常量时间算法 | 无通用防御；需逐指令类审计 |

---

### 【Slide N+10: WHEN】—— 关键时间线

```
2017  KAISER论文发表（KPTI前身）
  │
2018.01  Spectre (v1/v2) + Meltdown 公开披露 ← ★ 引爆点
2018.05  Intel发布IBRS/STIBP/IBPB白皮书
2018.07  Chrome 67默认启用站点隔离
2018.08  L1TF/Foreshadow 披露
2018.10  LLVM Speculative Load Hardening方案发布
  │
2019.05  MDS系列 (RIDL/Fallout/ZombieLoad) 披露
2019.08  Canella等发表瞬态执行攻击系统性综述
2019.11  TAA (TSX异步中止) 披露
2019      Intel Cascade Lake首次实现eIBRS + MDS/L1TF硬件修复
  │
2020.03  LVI (加载值注入) 披露，影响SGX
2020.06  SRBDS (特殊寄存器采样) 披露
2020      Linux Core Scheduling特性合入主线
  │
2021      Firefox首次默认启用Project Fission站点隔离
  │
2022.03  BHI (分支历史注入) 披露，eIBRS被绕
2022.07  Retbleed 披露，Intel/AMD返回指令再受影响
2022      Intel推出BHI_DIS_S硬件缓解（Alder Lake+）
2022      ARM宣布CSV2_3 (Cortex-X3) 最全面硬件隔离
  │
2023.07  Zenbleed 披露 (AMD Zen 2)
2023.08  GDS/Downfall 披露
2023.08  Inception 披露 (AMD Zen 3/4)
  │
2024.03  Native BHI (CVE-2024-2201) 无需eBPF
2024.10  ITS (Indirect Target Selection) 披露
2024.12  IBPD (Indirect Branch Predictor Delayed Updates) 披露
2024      Intel Granite Rapids发布IPRED/DDP硬件级防御
  │
2025-26  持续有新的瞬态执行变体研究发表
```

---

### 【Slide N+11: WHO】—— 关键角色图谱

```
     ┌─────────────────────────────────────────────────┐
     │                   攻击发现者                      │
     ├─────────────────────────────────────────────────┤
     │ Google Project Zero    │ Spectre, Meltdown,     │
     │                        │ Zenbleed               │
     │ ETH Zurich (ComSec)    │ Retbleed, Zenbleed     │
     │ VUSec (VU Amsterdam)   │ LVI, MDS系列           │
     │ TU Graz                │ Meltdown, KAISER       │
     │ UC San Diego           │ GDS/Downfall           │
     │ imec-DistriNet (KU Leuven) │ LVI, SGAxe         │
     └─────────────────────────────────────────────────┘

     ┌─────────────────────────────────────────────────┐
     │                   硬件防御方                      │
     ├─────────────────────────────────────────────────┤
     │ Intel    │ 30+微码更新，eIBRS/IPRED/BHI_DIS_S   │
     │ AMD      │ AGESA固件，PSFD，Inception缓解       │
     │ ARM      │ CSV2原语，SB指令，BTI，PAC            │
     │ RISC-V   │ 开放硬件验证，设计阶段纳管推测安全     │
     └─────────────────────────────────────────────────┘

     ┌─────────────────────────────────────────────────┐
     │                   软件防御方                      │
     ├─────────────────────────────────────────────────┤
     │ Linux内核    │ KPTI/Retpoline/VERW/Core Sched    │
     │ GCC/Clang    │ SLH/Retpoline推测屏障编译器支持    │
     │ KVM          │ VM边界清除+推测控制传递            │
     │ Xen/VMware/Hyper-V │ Hypervisor级防御             │
     │ Chrome V8    │ 站点隔离+JIT索引掩码              │
     │ Firefox      │ Project Fission站点隔离           │
     │ OpenSSL/BoringSSL │ 常量时间+推测屏障            │
     └─────────────────────────────────────────────────┘
```

### 【Slide N+12: WHERE】—— 防御技术应用到哪里？

| 应用场景 | 核心威胁 | 关键防御技术 | 防护成熟度 |
|----------|----------|-------------|-----------|
| **公有云 IaaS** (AWS EC2/Azure VM) | Guest→Host跨VM窃取 | KVM: L1D Flush + VERW + Core Scheduling | ★★★★ 高 |
| **私有数据中心** | 旧硬件累积性能损失 | 硬件换代评估；SMT禁用 vs Core Sched选择 | ★★★ 中-高 |
| **容器化平台** (Kubernetes/Docker) | 容器共享宿主机内核 | 继承宿主机KPTI/VERW；gVisor沙箱额外隔离 | ★★★ 中 |
| **Web浏览器** | JS→跨站/同进程数据窃取 | 站点隔离 + 定时器降级 + JIT推测屏障 | ★★★★ 高 |
| **移动端** (Android/iOS) | ARM旧型号未修复 | 依赖SoC厂商/Treble更新；应用层prctl | ★★ 中-低 |
| **IoT/嵌入式** | 老旧ARM内核无更新 | 几乎无有效防御；硬件隔离依赖少量新芯片 | ★ 低 |
| **机密计算** (Intel SGX/AMD SEV) | SGX飞地被推测攻击 | LLVM LVI缓解；SEV硬件隔离天然防御 | ★★★ 中 |
| **金融/密码学系统** | 密钥泄漏 | 常量时间算法 + 推测屏障 + prctl禁用推测 | ★★★★ 高 |
| **5G/边缘计算** | 多租户共享加速器 | GPU/NPU推测行为研究不充分 | ★★ 低 |

---

## 四、总结与建议页

### 洞察方向回顾

自2018年Spectre/Meltdown以来，瞬态执行漏洞已成为评估计算平台安全性的基础维度之一。这不仅是一场"漏洞发现→打补丁"的攻防循环，而是**催生了涵盖硬件设计、微架构规范、内核架构、编译器和语言运行时的全栈安全工程范式转变**。

### 价值识别

| 价值点 | 说明 |
|--------|------|
| **硬件换代决策依据** | 明确旧硬件(Skylake-)性能损失30-60% vs 新硬件(Granite Rapids/AMD Zen 5) <5% 的量化差距 |
| **安全架构选型参考** | ARM CSV2架构原语的设计前瞻性 vs x86打补丁模式的设计哲学对比 |
| **云厂商合规基线** | 所有主流云厂商已要求已修补硬件，成为基础设施准入门槛 |
| **纵深防御策略** | 单层防护（如仅KPTI）不足以应对全谱系攻击，需全栈协同 |
| **供应链安全** | AGESA固件更新、Linux内核版本、容器运行时均影响防御完备性 |

### 关键趋势

1. **硬件安全从"修复"走向"免疫"**：Intel IPRED、ARM CSV2_3、AMD Zen 4+代表推测隔离的硅片级硬化
2. **攻击发现从"一次性重大突破"转向"持续长尾"**：ITS、IBPD等2024年新变体证明新的微架构结构仍在成为攻击面
3. **RISC-V作为"安全重置"机会**：开放架构可以在设计阶段纳管推测安全，避免重蹈x86覆辙
4. **AI/ML加速器成为新攻击面**：GPU/NPU推测执行的侧信道研究仍处于早期阶段
5. **"安全性能"成为处理器关键指标**：Phoronix等基准测试已将安全缓解性能影响纳入CPU评测常态

### 技术识别

```
              新一代硬件 (2024+)
              ┌─────────────────────┐
              │  IPRED  │  CSV2_3   │
              │  DDP    │  BTI/PAC  │
              │  FSFP   │  Zen 5    │
              └────┬───┬──────┬───┬─┘
                   │   │      │   │
     ┌─────────────┼───┼──────┼───┼─────────────┐
     │  内核层      │   │      │   │             │
     │  KPTI/IBRS  │   │      │   │             │
     │  VERW/IBPB  │   │      │   │             │
     │  Core Sched │   │      │   │             │
     ├─────────────┤   │      │   ├─────────────┤
     │  编译器层    │   │      │   │  虚拟化层    │
     │  Retpoline  │   │      │   │  L1D Flush  │
     │  SLH/LFENCE │   │      │   │  VERW(VM)   │
     ├─────────────┤   │      │   ├─────────────┤
     │  浏览器层    │   │      │   │  密码学层    │
     │  站点隔离    │   │      │   │  常量时间    │
     │  JIT掩码    │   │      │   │  推测屏障    │
     └─────────────┘   │      │   └─────────────┘
                ┌──────┘      └──────┐
                │   IoT/嵌入式缺口    │
                │  （独立议题）       │
                └────────────────────┘
```

### 重要玩家

| 层级 | 关键组织 | 不可替代性 |
|------|---------|-----------|
| 硬件 | Intel / AMD / ARM / RISC-V国际 | 硅片级修复只能在设计阶段实现 |
| 发现 | Google P0 / ETH ComSec / VUSec / TU Graz | 独立安全研究驱动产业响应 |
| OS | Linux内核社区 / Microsoft / Apple | 操作系统是防御最后一道通用防线 |
| 云 | AWS / Azure / GCP | 推动硬件更新换代的核心商业力量 |
| 浏览器 | Google Chrome / Mozilla / Apple WebKit | 终端用户最大攻击面的防线 |
| 编译器 | GCC / LLVM / MSVC | 推测屏障从编译期嵌入，降低开发者负担 |
| 标准 | MITRE / NIST / ISO | CWE-1427等推动统一的瞬态执行安全标准 |

### 行动建议

1. **立即**：通过 `/sys/devices/system/cpu/vulnerabilities/` 评估当前部署环境风险状态
2. **短期（1年内）**：规划Skylake及更早硬件的替换；启用Core Scheduling（如使用SMT）；为敏感进程配置 `prctl()` 推测控制
3. **中期（2-3年）**：采购需求中纳入推测安全评估（IA32_ARCH_CAPABILITIES / CSV2 / ARM SB等）；容器化敏感应用使用gVisor等强隔离运行时
4. **长期（3-5年）**：关注RISC-V + CHERI等从设计阶段消除推测侧信道的下一代架构；推动组织建立瞬态执行安全基线（类似Common Criteria级别）

---

> **核心结论**：
>
> 微架构瞬态执行攻击不是单个可以被"修复"的漏洞，而是**现代推测执行处理器与生俱来的结构性矛盾**。全栈防御（硅片→微码→内核→编译器→运行时→浏览器）是当前唯一的工程答案，且这一防御栈仍在持续演进。**新硬件(2024+)已将性能损失控制在5%以内，显著改变了安全部署的经济账。** 真正的根本解决，需要从处理器设计的规格层面将"推测隔离"纳入第一性原理。

---
> *本文档基于截至2026年5月的公开信息编制。建议定期复查各厂商安全公告获取最新态势。*
