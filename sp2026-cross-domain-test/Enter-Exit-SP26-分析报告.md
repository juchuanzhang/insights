# 论文分析报告：《Enter, Exit, Page Fault, Leak：测试微架构泄漏的隔离边界》

---

## 一、论文基本信息

| 项目 | 内容 |
|---|---|
| **标题** | Enter, Exit, Page Fault, Leak‡ : Testing Isolation Boundaries for Microarchitectural Leaks |
| **作者** | Oleksii Oleksenko* (Azure Research/Microsoft), Flavien Solt* (ETH Zurich), Cédric Fournet (Azure Research/Microsoft), Jana Hofmann (MPI-SP), Boris Köpf (Azure Research/Microsoft), Stavros Volos (Azure Research/Microsoft) |
| **机构** | Microsoft Azure Research、ETH Zurich、MPI-SP |
| **开源工具** | Revizor v1.3+，https://github.com/microsoft/sca-fuzzer |
| **CVE编号** | CVE-2024-36357、CVE-2024-36350、CVE-2024-36349、CVE-2024-36348 |
| **AMD安全公告** | https://www.amd.com/en/resources/product-security/bulletin/amd-sb-7029.html |

> 标题中的"‡"引用了英国小说《Tinker, Tailor, Soldier, Spy》（锅匠、裁缝、士兵、间谍），隐喻一个MI6特工揭露向苏联泄密的内鬼——与本文揭示CPU微架构"内鬼泄漏"的主题呼应。

---

## 二、研究背景与动机

### 2.1 核心问题

现代计算机系统依赖**安全域**（security domain）抽象来隔离不同信任级别的执行环境——如OS内核与用户进程、hypervisor与虚拟机（VM）。虽然架构级隔离机制（页表、虚拟化、特权级）在逻辑层面提供了保护，但**共享微架构状态**（缓存、CPU缓冲区等）仍可能成为信息泄漏通道，典型例子包括Meltdown、Foreshadow、MDS等攻击。

### 2.2 当前困境

微架构防御的部署呈现出**碎片化**特征：

- **硬件/固件层**：部分泄漏由CPU硬件补丁缓解（如Meltdown/Foreshadow的硬件修复）
- **OS/hypervisor层**：部分由软件缓解（如缓冲区刷新、KPTI）
- **未修补**：部分因安全影响被认为较低、缓解成本较高而被忽略

这种碎片化导致：

1. 验证手段不足——当前要么依赖PoC攻击测试，要么依赖纯理论推理，均劳动密集且易出错
2. 补丁多次被证明不完整甚至完全无效（如Enhanced IBRS、FineIBT、retpoline、VERW等均被后续攻击绕过）
3. CPU厂商的安全声明屡被推翻

### 2.3 现有方案的局限

| 工具/方法 | 局限 |
|---|---|
| Revizor | 仅在内核态单一域内运行 |
| Scam-V | 仅检测ARM TrustZone单域泄漏 |
| SpeechMiner | 单域（用户态或内核态），不支持域间转换 |
| Transynther | 仅搜索Meltdown/MDS类已知漏洞变体，无法发现全新泄漏类型 |
| RegCheck | 仅针对Meltdown-3a特定类型 |

**核心差距**：没有任何工具能系统性、自动化地测试**跨域**微架构隔离。

---

## 三、核心方法论：基于Actor的模型关系测试（MRT）

### 3.1 Actor抽象

本文引入**Actor**作为安全域的统一抽象：

- Actor = 一段代码 + 一段私有数据 + 特定的特权级 + CPU模式 + 内存权限配置
- Actor之间通过预定义指令序列（如syscall、VM enter）进行域转换
- 每个test case包含一个"main" Actor（主机内核态）和一个或多个其他模式的Actor

### 3.2 MRT工作原理

MRT的核心思路是：**不直接比较硬件轨迹与模型预测，而是比较暴露的信息**。

测试流程五步循环：

```
1. 程序生成 → 随机汇编指令序列（模板+配置约束）
2. 输入生成 → 随机二进制数据初始化内存和寄存器
3. 硬件执行 → 在目标CPU上执行，收集硬件轨迹（如Flush+Reload）
4. 模型执行 → 在Unicorn模拟器上执行，收集合同轨迹（预期泄漏）
5. 泄漏比较 → 检查非干扰性质，过滤已知泄漏，报告未知泄漏
```

关键优势：即使模型很简单，也能有效过滤已知泄漏并检测未知泄漏——因为MRT比较的是**信息暴露量**而非轨迹本身。

---

## 四、技术架构详解

### 4.1 测试用例生成（§4）

#### 4.1.1 设计需求

两大核心需求：

1. **结构化（Structuring）**：域转换需要特定指令序列（如VMLAUNCH/VMRUN），缓解措施也需要特定序列（如VERW、L1D flush），因此测试用例不能完全随机，必须将固定序列嵌入随机上下文
2. **统一化（Unification）**：执行器（executor）和模型（model）两阶段需要不同的指令实例化（如模拟器不支持VM enter/resume指令），需要用统一方式描述但允许差异化实例化

#### 4.1.2 模板+宏机制

- **模板**：汇编文件，定义Actor结构和域转换，不同Actor代码放在不同section中
- **宏**：伪指令（汇编中表现为NOP），在不同阶段被实例化为不同操作
  - executor中：通过**二进制补丁**替换NOP为跳转到宏实现（如VMRESUME/VMRUN、Prime/Flush等）
  - model中：通过**动态回调**，在Unicorn模拟器中拦截NOP地址调用对应实现

```asm
; 模板示例（双Actor：main + guest）
.section .main
.start:
  .macro.random_instructions.64:   ; 随机指令区
  VERW qword ptr [r14]             ; 缓冲区刷新
  .macro.set_h2g_target.vm_start:  ; 设置VM入口
  .macro.set_g2h_target.end:       ; 设置VM出口
  .macro.switch_h2g:               ; 域转换宏
.end:
.macro.fault_handler:              ; 异常处理

.section .guest
.vm_start:
  .macro.measurement_start:        ; 开始测量
  .macro.random_instructions.64:   ; 随机指令区
  .macro.measurement_end:          ; 结束测量
  .macro.switch_g2h:               ; 返回host
```

#### 4.1.3 内存访问插桩

新增**插桩pass**：随机选择一个生成的内存访问，修改其地址指向其他Actor的内存，专门针对Meltdown类泄漏场景（将用户态Actor的内存访问指向内核态Actor的内存）。

### 4.2 执行环境（§5）

#### 4.2.1 设计需求

两大需求：

1. **完全可配置性**：自由配置页表权限、CPU模式、VM配置等，甚至允许无效系统配置以触发特定泄漏
2. **低开销**：目标≥100次测量/秒，需要在毫秒级创建VM和页表

#### 4.2.2 为什么不用Linux现有基础设施

- Linux的VM/页表管理面向完整VM，开销过大
- Linux内核有大量安全/稳定性防护，阻止自由配置实验
- 因此**从零实现**了3500+行代码的定制内核模块

#### 4.2.3 关键设计

```c
// 高层执行流程
traces = [];
disable_interrupts();            // 禁用中断
preserve_host_state();           // 保存host OS状态
configure_system_registers();    // 配置CPU（MSR、VMX/SVM等）
create_virtual_machines();       // 创建VM
create_page_tables();            // 创建页表
flush_caches_and_buffers();      // 刷新缓存

for (input in inputs)            // 对每个输入
    set_actor_data_memory(input);
    set_page_table_permissions();
    set_interrupt_descriptor_table();
    set_general_purpose_registers(input);
    htrace = execute(test_case); // 执行并收集轨迹
    traces.append(htrace);
    restore_page_table_permissions();
    restore_interrupt_descriptor_table();

restore_host_state();            // 恢复host OS状态
enable_interrupts();
return traces;
```

关键子设计：

| 子设计 | 说明 |
|---|---|
| **状态保存与恢复** | 保存/恢复CR0、EFER、VMX控制寄存器等，防止host OS崩溃 |
| **故障隔离** | 捕获所有异常→自定义处理器；简单异常→test case出口或fault_handler宏；NMI/machine check→恢复OS状态后退出executor |
| **CPU配置** | 通过配置文件控制MSR、性能计数器等（如禁用预取器） |
| **VM Actor配置** | 启用VMX/SVM→创建VMCS/VMCB→创建EPT/NPT和guest页表→设置权限 |
| **User Actor配置** | 更新系统进入/退出点指向user代码→更新页表权限允许用户访问 |
| **内存别名** | 可选功能：所有VM Actor使用相同虚拟地址布局，guest物理地址=host物理地址（类似Foreshadow-VM攻击条件） |

### 4.3 泄漏建模（§6）

#### 4.3.1 多Actor非干扰性质

经典非干扰：受害者数据σ^V不应影响攻击者测量Measure(p, σ, μ)

σ^1_A = σ^2_A ⇒ Measure(p, σ^1, μ) = Measure(p, σ^2, μ)

#### 4.3.2 合同式非干扰（实用化）

实际OS/hypervisor允许**基本泄漏**（如sysret后不刷新缓存→用户可观察内核load/store地址和跳转目标），但不允许MDS类更深层泄漏。

**Definition 1**：CPU满足隔离条件——对于任意程序p、任意两个架构状态σ^1和σ^2、任意初始微架构状态μ：

σ^1_A = σ^2_A ∧ Contract(p, σ^V_1) = Contract(p, σ^V_2) ⇒ Measure(p, σ^1, μ) = Measure(p, σ^2, μ)

**合同轨迹Contract(p, σ^V)** = 模型执行的预期泄漏信息（受害者load/store地址 + 控制流变化）

**隔离仅在泄漏超出内存访问地址和控制流变化时才被违反**。

#### 4.3.3 模型实现

在Unicorn模拟器中实现多域模型——**简化策略**：
- 所有Actor在用户态执行
- 域转换模拟为简单跳转
- 仅实现最小必要模拟（如VM exit对应特权指令）

### 4.4 非确定性处理（§7）

域转换（尤其VM enter/exit）涉及大量**不透明微码**，引入显著噪声。

#### 解决方案：从轨迹相等提升到分布相等

1. **问题**：两条硬件轨迹可能因噪声而不同（即使输入相同），导致大量假阳性
2. **核心思路**：将单次轨迹比较替换为**样本分布比较**
3. **统计工具**：Pearson χ²检验（将轨迹视为类别数据）

χ² = Σ (obs_t(t) − N·P(t))² / (N·P(t))

4. **自适应样本量**：从N=15开始→若检测到违反则逐步增大到40→160→320→仅当违反在最大样本量仍持续才报告

**效果**：检测已知泄漏的时间减少一个数量级，同时成功消除假阳性。

---

## 五、实验设计与结果

### 5.1 实验设计

#### 测试维度

| 维度 | 内容 |
|---|---|
| **隔离边界** | H2V（Host→VM）、V2V（VM→VM）、K2U（Kernel→User）、U2U（User→User） |
| **测试目标** | Int1(KabyLake R)、Int2(Coffee Lake)、Int3(RaptorCove)、AMD1(Zen1)、AMD2(Zen2)、AMD3(Zen4) |
| **配置类别** | MEM（内存错误）、COMP（计算错误/DSS）、REG（特权寄存器读取/RSRR） |

#### MEM配置变体

| 类型 | 错误类型 | 说明 |
|---|---|---|
| A-bit | assist | Accessed位为0 |
| D-bit | assist | Dirty位为0 |
| E/NPT A-bit | assist | EPT/NPT Accessed位为0 |
| E/NPT D-bit | assist | EPT/NPT Dirty位为0 |
| P-bit | #PF | Present位为0 |
| R-bit | #PF | Reserved位为1 |
| W-bit | #PF | Read/Write位为0 |
| E/NPT P-bit | #VMEXIT | EPT/NPT Present位为0 |
| E/NPT R-bit | #VMEXIT | EPT/NPT Reserved位为1 |
| E/NPT W-bit | #VMEXIT | EPT/NPT Write位为0 |
| U-bit | #PF | User位为0 |

#### 实验规模

- 304个独立测试campaign
- 88机器天
- 近3000万程序
- 超200亿硬件测量
- 仅2次假阳性

### 5.2 新发现（4个全新跨域泄漏）

#### ① 跨VM泄漏（CVE-2024-36357）——**高安全影响**

| 项目 | 详情 |
|---|---|
| **影响CPU** | AMD3 (Zen4) |
| **边界** | V2V |
| **配置** | MEM W-bit、MEM EPT W-bit |
| **机制** | 攻击者VM加载地址时，若缓存中包含受害VM相同虚拟地址（不同host物理地址）的值，受害者的缓存值影响攻击者后续操作的时序。攻击者通过时序差异推断受害VM缓存值的一位信息，通过移位操作可控制观察哪一位 |
| **攻击gadget** | 别名加载→两次写异常→两次非故障读→基于最后读取结果的缓存探测。gadget在异常时序与乱序执行代码之间创建竞争 |
| **根因** | 截至论文撰写时AMD尚未公开根因 |
| **影响评估** | **潜在高影响**——攻击者可逐位读取另一VM的任意内存 |
| **缓解** | AMD在论文embargo期间已开发并部署缓解措施 |

```c
// 跨VM泄漏gadget（伪代码）
a = *p1;           // p1别名指向受害VM地址
array1[a] = 0;
*p2 = 0;           // 页故障（写只读页）
...                 // 算术操作序列
*p3 = 0;           // 页故障（写只读页）
b = *p4;           // p4指向与p2/p3相同页
c = *p5;           // p5指向任意有效地址
...                 // 算术操作序列
d = array2[c];     // 缓存暴露点
```

#### ② 内核→用户泄漏（CVE-2024-36350）——**中等至高安全影响**

| 项目 | 详情 |
|---|---|
| **影响CPU** | AMD3 (Zen4) |
| **边界** | K2U |
| **配置** | MEM U-bit |
| **机制** | 用户进程可观察到最近N次（N≈32）内核态store操作的任意一位。攻击者可通过额外store选择目标store，通过移位选择泄漏位 |
| **gadget** | 类似跨VM泄漏gadget，但依赖用户态访问内核内存触发的异常 |
| **影响评估** | **中高影响**——攻击者可观察切换到用户态前的特权store数据 |
| **缓解** | AMD安全公告中提供 |

#### ③ RDTSCP-AUX特权读取（CVE-2024-36349）——**低安全影响**

| 项目 | 详情 |
|---|---|
| **影响CPU** | Int1, Int2, Int3, AMD1 |
| **边界** | K2U |
| **配置** | REG-Register |
| **机制** | 用户进程可推测性读取RDTSCP AUX寄存器，即使CR4.TSD标志位已设置（架构级应触发#GP故障） |
| **影响评估** | **低影响**——AUX寄存器通常仅含CPU ID，非敏感信息 |
| **重要发现** | AMD白皮书声称其CPU不受RSRR影响[29]，**本文结果直接反驳此声明** |
| **AMD态度** | 不计划修复（与Intel类似案例的决策一致） |

#### ④ SMSW推测执行（CVE-2024-36348）——**低安全影响但功能违反**

| 项目 | 详情 |
|---|---|
| **影响CPU** | AMD2 (Zen2), AMD3 (Zen4) |
| **边界** | K2U |
| **配置** | REG-UMIP |
| **机制** | AMD CPU在用户态推测性执行SMSW指令，即使UMIP功能已启用（架构级应触发#GP故障）。推测返回值为CR0寄存器的低16位 |
| **影响评估** | **低影响**——CR0低16位仅含CPU配置信息 |
| **重要发现** | **实质上绕过了UMIP功能**，且同样反驳AMD声称不受RSRR影响的声明 |
| **AMD态度** | 不计划修复 |

#### ⑤ Zen1 R/W位异常（待定）

在AMD1(Zen1)上发现R-bit和W-bit配置下的违规，但AMD在等效微架构/微码/BIOS/OS的CPU上无法复现，进一步调查暂不可行。

### 5.3 已知泄漏验证

工具成功检测了所有6类已知跨域泄漏：

| 已知攻击 | 配置 | 说明 |
|---|---|---|
| MDS及其变体 | MEM A/D-bit, EPT A/D-bit | ✓ |
| Foreshadow及其RMW变体 | MEM P-bit | ✓ |
| Meltdown | MEM U-bit | ✓ |
| DSS | COMP DSS | ✓ |
| Rogue System Register Read | REG FSGS | ✓（检测到变体Meltdown-3a） |

**关键结论**：在所有预期发现违规的实验中，均成功发现了违规——零漏检。

### 5.4 补丁有效性测试

| 补丁类型 | 测试结果 |
|---|---|
| **MDS - VERW** | ✓ 有效 |
| **MDS - L1D_FLUSH_CMD** | ✓ 有效 |
| **MDS - WBINVD** | ✗ 无效（非Intel推荐） |
| **Foreshadow - L1D_FLUSH_CMD** | ✓ 有效 |
| **Foreshadow - WBINVD** | ✗ 无效 |
| **DSS - 1÷1除法（当前Linux补丁）** | ✓ 有效 |
| **DSS - 异常处理器中的除法（旧补丁）** | ✗ 无效（差点合入主线内核！） |
| **Meltdown - KPTI（清除Present位）** | ✗ 无效 |
| **Meltdown - KPTI（完全unmapping）** | ✓ 阻止Meltdown，但仍有MDS违规 |
| **Meltdown - KPTI unmapping + VERW** | ✓ 有效 |

**特别发现**：DSS的旧补丁（将除法放在异常处理器而非系统进入例程中）被工具检测为**无效**——该补丁差点合入Linux主线内核，仅在安全社区审查后才被替换。这凸显了自动化验证工具的价值。

### 5.5 性能指标

| 指标 | 数据 |
|---|---|
| 测量速率 | 800–4500次/秒 |
| 测试吞吐量 | 60–700 test case/秒 |
| 完整campaign耗时 | MEM: 2–24h, COMP: 3–8h, REG: 1.5–11h |
| 违规检测时间 | 85%在1h内, 98%在4h内 |
| 按test case计 | 85%在20K轮内, 93%在40K轮内 |
| 漏检率 | 4%（10次不同seed重跑） |

---

## 六、局限性与未来方向

### 6.1 工具局限

| 局限 | 说明 |
|---|---|
| 不支持Actor间共享内存 | 架构级信息流（如共享内存通信）未被覆盖 |
| 不支持自修改代码 | — |
| 控制结构位置固定 | IDT/GDT位置固定，无法检测CPU对控制结构位置的预测泄漏 |
| 测量框架干扰 | 宏代码序列会影响微架构状态（但通过大量测试campaign和最小化设计缓解） |
| 仅覆盖泄漏提取型攻击 | 未覆盖跨域分支训练等控制流攻击 |
| 仅部分泄漏场景 | 某些检测需要大量测量，实验时长可能不足以发现所有可能泄漏 |

### 6.2 未来方向

1. 实现共享内存支持、自修改代码支持、控制结构位置随机化
2. 覆盖注入型泄漏（LVI、Spectre V2）——通过逆向模板（攻击者代码注入→切换到受害者→监测受害者轨迹）
3. 移植到ARM、RISC-V等其他架构
4. 支持其他隔离抽象（如Intel MPK）
5. 作为PoC原型工具——安全研究者可用手动模板快速构建攻击PoC（如Foreshadow仅需<30行模板汇编）

---

## 七、对微架构安全领域的意义与评价

### 7.1 方法论突破

| 方面 | 评价 |
|---|---|
| **从单域到跨域** | 首个系统性、自动化测试跨域微架构隔离的工具，填补了关键空白 |
| **从被动补丁验证到主动安全验证** | 推动范式转变——从"攻击出现→匆忙修补"转向"持续验证→主动发现" |
| **Actor抽象** | 统一且灵活——可表示VM、进程、内核等任何安全域 |
| **模板+宏机制** | 兼顾结构化（域转换/缓解序列）与随机性（探索大量微架构条件） |
| **统计方法处理非确定性** | χ²检验+自适应样本量——优雅解决域转换引入的不透明微码噪声问题 |

### 7.2 实验成果突破

- **4个全新CVE**（2个高影响、2个低影响但反驳厂商声明）
- **反驳AMD白皮书声明**——AMD声称不受RSRR影响，本文在AMD CPU上发现了两类RSRR（RDTSCP-AUX和SMSW推测执行）
- **发现无效补丁**——DSS旧补丁被检测为无效（差点合入Linux主线）；KPTI仅清除Present位不能阻止Meltdown
- **极低假阳性率**——304个campaign仅2次假阳性
- **零漏检已知攻击**——在所有预期发现违规的配置中均成功检测

### 7.3 实践价值

1. **CPU厂商验证工具**：可系统性验证微架构隔离声明和补丁有效性
2. **安全研究者PoC平台**：提供稳定低噪声环境，<30行汇编即可演示Foreshadow
3. **开源可复现**：工具作为Revizor v1.3+开源，降低了微架构安全验证门槛

### 7.4 与相关工作的核心差异

| 工具 | 域范围 | 漏洞类型 | 能否发现新类型 | 能否跨域 |
|---|---|---|---|---|
| **本文工具** | 多域（VM/User/Kernel） | 通用 | ✓ | ✓ |
| Revizor | 单域（内核） | 通用 | ✓ | ✗ |
| Scam-V | 单域（TrustZone） | 通用 | ✓ | ✗ |
| SpeechMiner | 单域 | Meltdown类定量 | ✗ | ✗ |
| Transynther | 双线程 | Meltdown/MDS变体 | ✗ | ✗ |
| RegCheck | 单域 | Meltdown-3a专用 | ✗ | ✗ |
| Osiris/Plumber | 单域 | 时序侧信道 | 部分 | ✗ |

### 7.5 批评性思考

**优势**：
- 方法论设计严谨，Actor抽象+模板+宏+合同式非干扰+χ²统计，形成完整闭环
- 实验规模庞大（88机器天、200亿测量），覆盖6款CPU、4种边界、3类配置
- 实用性强——既可自动化fuzzing，也可手动写PoC模板

**不足/可质疑**：
- 模型过于简化——所有Actor在用户态模拟、域转换仅为跳转，是否遗漏某些域转换特有的微架构行为？
- 未覆盖Spectre V2类跨域分支训练攻击——这是一个重要的攻击类别
- 测试仅限x86——ARM/RISC-V移植尚未实现
- 某些检测需要大量测量——4%漏检率表明某些泄漏可能被遗漏
- AMD1的R/W位违规AMD无法复现——可能受特定硬件配置影响，工具可重复性存疑

---

## 八、关键结论总结

1. **微架构隔离验证亟需自动化工具**——当前被动式补丁验证实践屡次失败
2. **Actor抽象+MRT方法**提供了系统性跨域微架构泄漏检测的可行路径
3. **四大新发现**（CVE-2024-36357/36350/36349/36348）证明了工具的有效性，其中两个具有较高安全影响
4. **AMD的RSRR安全声明被实证反驳**——在AMD CPU上发现了两类RSRR漏洞
5. **补丁验证价值显著**——发现DSS旧补丁无效（差点合入Linux主线）、KPTI多种变体的有效性差异
6. **从被动补丁到主动验证的范式转变**是处理器安全设计的未来方向

---

## 九、术语对照表

| 英文术语 | 中文翻译 | 说明 |
|---|---|---|
| Security Domain | 安全域 | 逻辑隔离的执行环境 |
| Microarchitectural Leak | 微架构泄漏 | 通过共享微架构状态（缓存/缓冲区）的信息泄漏 |
| Model-Based Relational Testing (MRT) | 模型关系测试 | 基于模型的随机测试方法论 |
| Actor | Actor（执行者） | 安全域的统一抽象 |
| Template | 模板 | 定义Actor结构和域转换的汇编文件 |
| Macro | 宏 | 可在不同阶段实例化的伪指令 |
| Contract Trace | 合同轨迹 | 模型预测的预期泄漏信息 |
| Hardware Trace | 硬件轨迹 | 实际CPU执行观测到的微架构状态变化 |
| Noninterference | 非干扰性质 | 安全隔离的形式化定义 |
| Domain Transition | 域转换 | 安全域之间的执行上下文切换 |
| VMCS/VMCB | 虚拟机控制结构 | Intel VMCS / AMD VMCB |
| EPT/NPT | 扩展/嵌套页表 | Intel EPT / AMD NPT |
| MDS | 微架构数据采样 | Microarchitectural Data Sampling |
| DSS | 除法器状态采样 | Divider State Sampling |
| RSRR | 系统寄存器特权读取 | Rogue System Register Read |
| UMIP | 用户模式指令预防 | User Mode Instruction Prevention |
| KPTI | 内核页表隔离 | Kernel Page Table Isolation |

---

## 十、FAQ（答疑记录）

### Q1：为什么作者不做跨TEE-REE的测试？

**问**：为什么作者不测试Intel TDX和AMD SEV-SNP这种CVM（Confidential VM）边界？

**答**：基于论文内容分析，不测TDX/SEV-SNP的原因主要有以下几点：

**1. Actor模式不支持**

论文脚注明确：当前仅支持 **host-kernel、host-user、guest-kernel** 三种组合。而TDX引入了全新的**SEAM模式**（SEAM-VMX non-root和SEAM-VMX root），这是现有框架完全没有覆盖的CPU特权级。

**2. 信任模型完全相反**

- 论文的V2V模板：hypervisor是**可信的**，它管理VM切换并收集测量——测试的是VM之间的泄漏
- TDX/SEV-SNP：hypervisor是**不可信的**，VM要隔离**FROM** hypervisor——这意味着需要完全不同的模板设计（攻击者是hypervisor，受害者是CVM）

**3. 执行器缺少关键基础设施**

- TDX需要TDX模块运行在SEAM模式、SEAM相关VMCS配置、TDCALL/TDXVMCALL指令处理——当前executor仅支持标准VMX/SVM
- SEV-SNP需要RMP表管理、SEV-SNP特定VMCB配置、RMPADJUST/PVALIDATE等指令的完整域转换序列——当前executor无此能力

**4. 硬件可用性问题**

论文测试时间为2024年，TDX当时仅在少数云平台试点部署（需BIOS启用、特定CPU微码），SEV-SNP虽在Zen3+支持但CVM完整设置流程复杂。6台测试CPU中仅AMD3(Zen4)理论上可运行SEV-SNP，但无TDX硬件。

**5. REG配置中已触及但未深入**

附录A.2的SEC类指令已包含SEV-SNP相关指令（PSMASH、PVALIDATE、RMPADJUST、RMPUPDATE），但这些仅作为**特权寄存器读取**（RSRR）场景的一部分——即测试用户态推测执行这些指令会泄漏什么，而非测试CVM→hypervisor的完整域转换泄漏。

**6. 内存别名机制的局限**

论文的内存别名功能让所有VM共享相同虚拟地址布局，模拟Foreshadow-VM条件。但SEV-SNP的RMP表硬件级阻止了hypervisor访问CVM内存，TDX的EPT权限限制也不同——需要全新的别名策略。

**总结**：不是"不想测"，而是测TDX/SEV-SNP需要**新增Actor模式、逆向信任模型模板、CVM执行器基础设施、CVM硬件平台**——这是一个工作量与论文本身相当的独立研究方向。

---

### Q2：缓解措施也需要特定序列（如VERW、L1D flush），怎么理解？

**问**：论文说"缓解措施也需要特定指令序列"，具体怎么理解？

**答**：很多微架构隔离缓解措施本身是**特定指令序列**，必须在域转换的**特定位置**执行才能生效，而不是一个"开关"一劳永逸。

**1. 缓解措施是指令序列，不是配置位**

- **VERW**：执行`VERW qword ptr [r14]`指令，刷新微架构store buffer（缓解MDS）。不是设置某个CPU配置位，而是**每次域转换前必须实际执行这条指令**
- **L1D flush**：执行`L1D_FLUSH_CMD` MSR写入指令，刷空L1数据缓存（缓解Foreshadow）。同样需要每次手动触发
- **DSS补丁**：Linux在切换到用户态前执行`DIV 1, 1`（一次假除法运算），刷新除法器状态。这是一条**插入在syscall退出路径中的指令**
- **KPTI**：需要在切换到用户态前**修改页表项**（一系列内存写操作），切换回来后再恢复

**2. 为什么模板需要包含它们**

测试的目标是验证"这个缓解措施是否真的有效"。因此模板中需要：

```asm
.section .main
.start:
  .macro.random_instructions.64:   ; 受害者随机代码
  VERW qword ptr [r14]             ; ← 缓解措施：在域转换前刷store buffer
  .macro.switch_h2g:               ; ← 域转换：切换到攻击者
.end:
```

如果模板中**没有VERW**，测到的泄漏可能是"没有缓解时的泄漏"——但真正关心的是**"加了缓解后还漏不漏"**。

**3. 缓解序列的位置很关键**

缓解措施必须在**域转换的紧前方**执行才有意义——VERW刷完store buffer后立刻切换到攻击者Actor，攻击者才无法从buffer中读到受害者数据。如果VERW和域转换之间插入大量随机指令，buffer可能又被填充，缓解就失效了。

这就是为什么模板设计是**结构化的**：固定序列（缓解+域转换）必须保持顺序，随机代码只能在固定序列之外的"空位"中填充。

**4. 同一缓解在不同场景下的实例化不同**

同一缓解措施在不同场景下的具体实现可能不同——比如测试"VERW能否阻止VM→VM的MDS泄漏"时，VERW放在Host Actor中、在VM enter之前；而测试"VERW能否阻止Kernel→User的MDS泄漏"时，VERW放在Kernel Actor中、在sysret之前。模板机制允许**用同一个模板框架描述不同场景**，通过宏配置灵活插入缓解序列。

---

### Q3：论文测了哪些防护措施？

**问**：论文具体测试了哪些微架构隔离防护措施？

**答**：论文测试的防护措施分为两类：**微架构隔离补丁**（§8.2.3）和**工具自身的防噪声/防干扰措施**。

**一、微架构隔离补丁（核心测试内容）**

MDS补丁：

| 补丁 | 测试结果 | 说明 |
|---|---|---|
| **VERW指令** | ✓ 有效 | Intel推荐，每次域转换前刷store buffer |
| **L1D_FLUSH_CMD** | ✓ 有效 | Intel推荐，刷L1数据缓存 |
| **WBINVD** | ✗ 无效 | 非Intel推荐，功能类似缓存刷新但无法阻止MDS泄漏 |

Foreshadow补丁：

| 补丁 | 测试结果 |
|---|---|
| **L1D_FLUSH_CMD** | ✓ 有效 |
| **WBINVD** | ✗ 无效 |

DSS补丁（除法器状态采样）：

| 补丁 | 测试结果 | 说明 |
|---|---|---|
| **当前Linux补丁（1÷1除法）** | ✓ 有效 | 在sysret前执行假除法，刷新除法器状态 |
| **旧补丁（异常处理器中的除法）** | ✗ **无效** | 将除法放在#DE异常处理器而非系统进入例程中——**差点合入主线内核！**仅经安全社区审查后才被替换 |

Meltdown补丁（KPTI）：

| KPTI变体 | 测试结果 | 说明 |
|---|---|---|
| **清除Present位** | ✗ **无效** | 仅标记内核页为无效，仍可Meltdown读取 |
| **完全unmapping（零化PTE）** | ✓ 阻止Meltdown | 但仍检测到user→kernel访问触发MDS的违规 |
| **unmapping + VERW** | ✓ **有效** | 完全unmapping配合VERW才能同时阻止Meltdown和MDS |

**二、工具自身的防噪声/防干扰措施**

| 控制措施 | 说明 |
|---|---|
| **禁用中断** | executor执行期间禁用核心所有中断，防止意外VMEXIT/上下文切换 |
| **刷缓存和缓冲区** | 每次测量前刷新caches和buffers，确保干净的微架构起始状态 |
| **保存/恢复host状态** | 实验前后保存恢复CR0、EFER、VMX控制寄存器等，防止host OS崩溃 |
| **禁用预取器** | 配置文件中`enable_prefetchers: false`，通过MSR禁用CPU预取器减少噪声 |
| **LFENCE防直行推测** | 宏跳转后插入LFENCE，防止straight-line speculation干扰宏实现 |
| **保留寄存器子集** | 测量代码预留部分寄存器避免额外内存访问，最小化对微架构状态的干扰 |

**三、汇总**

论文共测试了**7种防护补丁变体**（VERW、L1D_FLUSH_CMD、WBINVD×2、DSS当前补丁、DSS旧补丁、KPTI×3），其中发现**3种无效**（WBINVD×2、DSS旧补丁、KPTI-清除Present位），这恰好证明了论文核心论点——**没有自动化验证，你无法确认补丁真的有效**。