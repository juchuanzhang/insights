# 会话上下文记录

> **创建日期**: 2026-06-02  
> **用途**: 跨设备/跨session恢复工作上下文

---

## 项目概述

**项目名称**: 微架构瞬态执行漏洞防护机制洞察  
**项目目录**: `microarchitecture-attack-defense-new/`  
**所属仓库**: `https://github.com/juchuanzhang/insights.git` (branch: main)  
**最新commit**: `05e34ff` - 新增v2.0文档(覆盖Windows/macOS/MSVC等商用平台)

---

## 已完成工作

### 2026-06-02 Session

1. 在 `microarchitecture-attack-defense-new/` 目录下创建了 `微架构瞬态执行漏洞防护机制洞察.md` (v2.0, 1795行)
2. 已 git commit 并 push 到 GitHub

**文档结构 (15章)**:
- 1 引言
- 2 瞬态执行攻击分类体系
- 3 Intel 商用处理器防御机制 (IA32_SPEC_CTRL / eIBRS / BHI_DIS_S / IPRED / VERW / DDP / FSFP)
- 4 AMD 商用处理器防御机制 (IBRS / Auto-IBRS / PSFD / Zenbleed / Inception / LFENCE序列化)
- 5 ARM 商用处理器防御机制 (CSV2系列 / SB / SSBD / BTI / PAC / MTE)
- 6 Windows 操作系统防御机制 (KVA Shadow / Retpoline+IBRS / SetProcessMitigationPolicy / SSBD / MDS/TAA / L1TF / Get-SpeculationControlSettings)
- 7 Linux 内核防御机制 (KPTI / Retpoline / RSB填充 / SSBD / L1D Flush / VERW清除 / sys漏洞状态 / prctl)
- 8 macOS 操作系统防御机制 (Double Map / Apple芯片硬件修复 / PAC/BTI)
- 9 编译器级防御机制 (GCC / Clang/LLVM / MSVC `/Qspectre`系列 / ARM Compiler / 内置屏障函数汇总)
- 10 虚拟化平台防御机制 (KVM / Xen / VMware ESXi / Hyper-V / Apple Hypervisor.framework)
- 11 Web 浏览器防御机制 (Chrome/Firefox/Safari/Edge: SharedArrayBuffer限制/定时器降级/站点隔离/JIT缓解/Wasm防护)
- 12 其他关键商用软件防御机制 (数据库/密码学库/容器运行时/编程语言运行时/云服务商)
- 13 防护效能对比分析 (硬件成熟度/OS覆盖度/编译器覆盖度/性能影响/综合防护等级)
- 14 结论与展望
- 15 参考文献 (115条)

---

## 关键决策记录

1. **v2.0 vs v1.0 差异**: v2.0 在 v1.0 基础上新增了:
   - 第6章 Windows 操作系统防御机制（全新专章）
   - 第8章 macOS 操作系统防御机制（全新专章）
   - 第9章编译器部分新增 MSVC `/Qspectre` 系列选项和 ARM Compiler 6 专节
   - 第10章新增 Apple Hypervisor.framework
   - 第12章新增 SQL Server、Windows CNG、Windows Containers、Swift
   - 第13章新增 OS防御覆盖度对比、编译器防御覆盖度对比两个子节
   - 第4章新增 Auto-IBRS 和 LFENCE序列化语义两个子节
   - 参考文献从 ~110 条扩展至 115 条

2. **用户要求**: 忽略当前文件夹以外的所有文件，仅在工作目录 `microarchitecture-attack-defense-new/` 内操作

---

## 待办/后续可能工作

- [ ] 添加 PPT 提纲文件（如 v1.0 中有）
- [ ] 针对特定新CVE补充细节
- [ ] 性能数据标注存疑项的核实与修正
- [ ] 新发现的瞬态执行漏洞跟踪更新
- [ ] 绘制防御架构图/流程图

---

## 仓库结构

```
insights/                                          (git repo root)
├── README.md
├── LICENSE
├── microarchitecture-attack-defense/               (v1.0, 已有)
│   ├── 微架构瞬态执行漏洞防护机制洞察.md
│   └── 微架构瞬态执行漏洞防护-PPT提纲.md
└── microarchitecture-attack-defense-new/           (v2.0, 当前工作目录)
│   ├── 微架构瞬态执行漏洞防护机制洞察.md
│   └── SESSION_CONTEXT.md                          (本文件)
```

---

## 快速恢复指令

```bash
# 查看当前状态
git log --oneline -3

# 继续编辑文档
# 文件路径: microarchitecture-attack-defense-new/微架构瞬态执行漏洞防护机制洞察.md

# 提交并推送
git add microarchitecture-attack-defense-new/
git commit -m "描述变更内容"
git push origin main
```