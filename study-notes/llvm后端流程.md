## 总指挥

`TargetPassConfig` 是**后端阶段的配置者**：它决定后端阶段要加哪些 Pass。

> Tips1：`TargetPassConfig` 并不负责**调度和执行** Pass，那是 `PassManager`的工作。`TargetPassConfig`是配置者，它利用 `PassManager` 的 `add()` 接口，把后端 Pass 注册进去，可以理解为一个配置脚本。
>
> Tips2：`TargetPassConfig` 也是一个pass，是一个特殊的pass，`TargetPassConfig` **继承自 `ImmutablePass`**，它也会被加入 `PassManager`。

### `TargetPassConfig`和`PassManager`

谁把 `PassManager` 传给 `TargetPassConfig`？

在编译驱动（如 `llc`）或 `TargetMachine::addPassesToEmitFile()` 里，流程大致是：

```c++
// 1. 创建 PassManager
legacy::PassManager PM;

// 2. 创建 TargetPassConfig，并把 PM 的引用传进去
TargetPassConfig *TPC = TM.createPassConfig(PM);

// 3. 把 TPC 自己加入 PM（TPC 本身也是一个 ImmutablePass）
PM.add(TPC);

// 4. 调用 TPC 的配置方法，让它往 PM 里加后端 Pass
TPC->addMachinePasses();

// 5. 启动 PassManager，按顺序执行所有 Pass
PM.run(Module);
```

### `TargetPassConfig`怎么注册后端pass

`TargetPassConfig` 自己定义了一个 `addPass()` 方法，它本质上就是对 `PM.add()` 的封装：

```c++
void TargetPassConfig::addPass(Pass *P) {
  PM.add(P);
}
```

大致结构：

```c++
void TargetPassConfig::addMachinePasses() {
  addISelPasses();          // 内部用 addPass(...) 添加 SelectionDAGISel 或 FastISel
  addOptimizedRegAlloc();   // 内部添加 RAGreedy 等
  // ...
}
```

TargetPassConfig::addISelPasses() 中：（示意代码）

```c++
if (UseFastISel)
  addPass(createFastISelPass(...));
else
  addPass(createSelectionDAGISelPass(...));
```

## 基于SelectionDAG的指令选择

**SelectionDAGISel** 这条管线做了“把 IR 基本块转成 DAG，再合法化、combine、指令选择、调度、形成 MachineInstr”这一系列操作；SelectionDAG 是它使用的 DAG 表示。

**SelectionDAGISel** 是一个大的pass，是总控入口。

```text
经过opt的LLVM IR
	|
	| SelectionDAGBuilder，初步转换DAG
	|
初始的DAG
	|
	| 这些合法化和优化操作也在SelectionDAGISel里调用
	| 类型合法化 (LegalizeTypes)、DAG Combine (优化，被多次调用)、操作合法化 (LegalizeDAG)
	|
目标无关的合法化 DAG
	|
	| 模式匹配
	|
目标相关的 Machine DAG
	|
	| 指令调度
	|
线性 MachineInstr 序列（每个基本块对应一个 MachineBasicBlock，里面是 MachineInstr 列表）
```

### SelectionDAGBuilder

`SelectionDAGBuilder` 是一个**目标无关的 lowering 实现**，它由 `TargetLowering` 对象进行参数化。它的主要职责是**遍历 LLVM IR 的基本块，并将其中的指令转换为 `SDNode`，从而构建出初始的 `SelectionDAG`**。

**`TargetLowering` **：

- `SelectionDAGBuilder` 持有一个 **`TargetLowering` 的引用/指针**。`TargetLowering` 是一个抽象基类，每个目标后端都实现一个子类。
- `SelectionDAGBuilder` 在需要目标信息时，就通过这个 `TargetLowering` 接口去询问。

**在流程中的位置**：

- `SelectionDAGBuilder` 通常由 `SelectionDAGISel` 这个 Pass 驱动。`SelectionDAGISel` 会为每个基本块创建 `SelectionDAGBuilder` 的实例，并调用它来构建初始 DAG，之后才会依次执行类型合法化、DAG Combine 和指令选择等后续步骤。

### 模式匹配

该阶段将采用目标无关的DAG节点作为输入，匹配特定的模式，将其映射到特定平台的DAG输出节点（Machine DAG）。

主要类：

- SelectionDAGISel是在SelectionDAG的基础上，进行基于模式匹配的指令选择器的通用基类。
- TableGen类帮助选择特定平台的指令。

CodeGenAndEmitDAG()函数调用DoInstructionSelection()函数，遍历DAG节点并对每个节点调用Select()函数，如下：

```c++
SDNode *ResNode = Select(Node);
```

Select()函数是需要由特定平台实现的抽象方法。x86目标平台实现了**X86DAGToDAGISel::Select()**函数。这个函数拦截一部分节点进行手动匹配，而大部分工作则委托给**X86DAGToDAGISel::SelectCode()**函数完成。

X86DAGToDAGISel::SelectCode函数由TableGen自动生成。它包含一个匹配表，随后调用SelectionDAGISel::SelectCodeCommon()泛型函数，并把匹配表传给它。

相关命令：

```bash
// 查看指令选择之前的DAG
llc –view-isel-dags test.ll

// 查看指令选择之后的DAG
llc –view-sched-dags test.ll
```

关于指令选择的具体实现，可以阅读源码lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp文件。

### 指令调度

经过模式匹配后，我们的Selection DAG节点已经变成由目标平台支持的指令和操作数组成的Machine DAG了。但是，目标架构以序列执行指令。所以，下一步是指令调度，即线性化DAG，输出指令序列到MachineBasicBlock。

主要类：

- ScheduleDAG是调度器基类，被其它调度器类继承。

  - ```c++
    ScheduleDAG
        ↑
    ScheduleDAGSDNodes
        ↑
    ScheduleDAGList / ScheduleDAGRRList / ScheduleDAGFast ...
    ```

- **`SelectionDAGISel` 负责创建调度器（通常是 `ScheduleDAGRRList` 或 `ScheduleDAGList`），并调用其 `Run` 方法**。

调度器（`ScheduleDAGList`、`ScheduleDAGRRList` 等）具体做的事情，可以概括为**三个大阶段**：构建调度图、列表调度、发射机器指令。

#### 构建调度图

调度器首先把模式匹配后得到的 **Machine DAG** 转换成一张**调度图**（Schedule DAG）。张图的核心元素是：

- **`SUnit`**：调度单元，对应 DAG 中的一个 `SDNode`（通常是一条机器指令）。
- **`SDep`**：调度依赖边，描述 `SUnit` 之间的约束关系。

这一步由 `ScheduleDAGSDNodes::BuildSchedGraph` 完成。

#### 列表调度

列表调度决定了**指令的线性顺序**，是调度器的**核心算法**，有多个调度器类都实现了列表调度。

以 `ScheduleDAGRRList` 为例，大致流程是：

1. **计算优先级**
   - 对每个 `SUnit` 计算一个优先级，通常基于**关键路径长度**（从该节点到出口的最长延迟路径）。
   - 关键路径上的节点优先级最高，因为它们决定了整体执行时间。
2. **维护就绪队列**
   - 一个 `SUnit` 只有在所有前驱依赖都满足后，才进入就绪队列。
   - 队列按优先级排序，优先取出优先级最高的节点。
3. **调度循环**
   - 从就绪队列取出一个节点，安排为下一条指令。
   - 更新其后继节点的依赖计数，把新就绪的节点加入队列。
   - 重复直到所有节点都被调度。
4. **寄存器压力控制**（`ScheduleDAGRRList` 的特色）
   - 在每一步，调度器会评估当前**活跃虚拟寄存器的数量**。
   - 如果某个选择会导致寄存器压力过高，可能触发**溢出（spill）**或选择压力更低的节点。
   - 这就是 `RR`（Register Reduction）的含义：在调度时尽量减少寄存器压力。

调度方向可以是：

- **自顶向下（Top-Down）**：从入口向出口调度。
- **自底向上（Bottom-Up）**：从出口向入口调度。

#### 发射机器指令

调度完成后，调度器按确定的顺序遍历 `SUnit`，把每个 `SUnit` 对应的 `SDNode` 转成 `MachineInstr`，插入到 `MachineBasicBlock` 中。

这一步由 `ScheduleDAGSDNodes::EmitSchedule` 完成。

发射完成后，`SelectionDAG` 被销毁，后续流程（寄存器分配等）就基于这个线性的 `MachineInstr` 序列进行。从这一刻起，后端流水线进入 **MachineFunction 阶段**。

```text
MachineFunction
  └── MachineBasicBlock (多个)
        └── MachineInstr (多条，双向链表)
```

## 寄存器分配

前一步执行完后，`MachineInstr` 列表的指令中的操作数使用的是**虚拟寄存器**。

寄存器分配的任务就是为虚拟寄存器分配物理寄存器。在LLVM 中，虚拟寄存器的数量是无限的，而**物理寄存器的数量是有限的**，根据目标架构确定。这些有限的寄存器需要被有效分配。如果无法做到，就会造成寄存器溢出。因此，通过寄存器分配，我们的目的是将最多的物理寄存器分配给虚拟寄存器。

> 寄存器溢出：当物理寄存器不足以容纳所有活跃的变量时，部分变量会被“溢出”（Spill）到栈内存中。

### TableGen

TableGen 是一个元编程工具，它读取后缀为 `.td` 的目标描述文件，自动生成 C++ 代码（通常为 `.inc` 文件）。这些代码会被编译进 LLVM 后端，为寄存器分配等阶段提供必要的数据结构。

### TD文件

对于任何目标芯片，你都需要在 TD 文件中定义其寄存器信息，通常位于 **`XXXRegisterInfo.td`** 文件（如 `X86RegisterInfo.td`）中。关键定义包括：

- **寄存器本身**：定义每个物理寄存器（如 `AL`, `AX`, `EAX`, `RAX`）的名称、编码及其别名关系。
- **寄存器类 (RegisterClass)**：将具有相同属性（如都是 32 位通用寄存器）的寄存器分组。**寄存器类同时定义了寄存器分配的默认顺序**，分配器会按此顺序尝试分配物理寄存器。
- **溢出大小 (SpillSize)**：指定寄存器被溢出到栈上时所需的大小（以位为单位），如果为 0，TableGen 会从寄存器类中推断。

以 x86 为例，当 LLVM 编译 x86 目标时，TableGen 会处理 `X86RegisterInfo.td` 等文件，并生成 `X86GenRegisterInfo.inc`。寄存器分配器的工作直接依赖于这个生成的文件。

## 代码发射

MC 层负责将上一步传送过来的MachineInstr序列变成汇编文件`.s`或者目标文件`.o`。

具体流程大致是：

```text
MachineInstr 序列
  -> AsmPrinter 等转成 MCInst（lowering）
  -> MCCodeEmitter 编码成目标机器码字节
  -> MCStreamer 写出
       - MCAsmStreamer   -> .s 汇编
       - MCObjectStreamer -> .o 目标文件
```

也就是说，LLVM 后端代码发射负责的是：`MachineInstr -> MCInst -> 机器码 -> 汇编/目标文件`。而可执行文件还需要链接器的参与。

### AsmPrinter

目标平台需要实现继承AsmPrinter的子类：在目标的代码目录下（如 `lib/Target/XXX/`）新建 `XXXAsmPrinter.cpp` 文件，并定义一个继承自 `llvm::AsmPrinter` 的类。

举个例子：X86AsmPrinter调用X86MCInstLower类，执行目标相关的lowering操作。

### MCCodeEmitter

`MCCodeEmitter` 是 **MC 层负责把 `MCInst` 编码成目标机器码字节** 的组件。它只在需要生成二进制机器码时起作用，比如生成可重定位目标文件 `.o` 或 JIT 执行；如果只是生成汇编文本 `.s`，则不需要它。

同样，需要实现`XXXMCCodeEmitter`。

### MCStreamer

现在，我们已经得到MCInst 指令，将它们输送给MCStream 类进一步处理，以生成汇编文件或者目标代码。根据不同选择，MCStreamer 用它的子类MCAsmStreamer 生成汇编代码，或者用它的子类MCObjectStreamer 生成目标代码。

- MCAsmStreamer 调用目标特定的 MCInstPrinter 打印汇编指令。
- MCObjectStreamer 使用LLVM 目标代码汇编器生成二进制代码。这个汇编器进而调用`MCCodeEmitter::EncodeInstrction()`生成二进制指令。

### 代码目录

```
llvm/
├── include/llvm/
│   ├── MC/
│   │   ├── MCStreamer.h            # MCStreamer 基类接口
│   │   ├── MCAsmStreamer.h         # 汇编文本 Streamer 接口
│   │   ├── MCObjectStreamer.h      # 目标文件 Streamer 接口
│   │   └── MCCodeEmitter.h         # MCCodeEmitter 基类接口
│   └── CodeGen/
│       └── AsmPrinter.h            # AsmPrinter 基类接口（位于 CodeGen 下）
│
├── lib/
│   ├── MC/                         # ✅ 所有通用 MC 实现
│   │   ├── MCStreamer.cpp          # MCStreamer 基类实现
│   │   ├── MCAsmStreamer.cpp       # 汇编文本 Streamer 实现
│   │   ├── MCObjectStreamer.cpp    # 目标文件 Streamer 基类实现
│   │   ├── MCELFStreamer.cpp       # ELF 目标文件 Streamer
│   │   ├── MCMachOStreamer.cpp     # Mach-O 目标文件 Streamer
│   │   ├── WinCOFFStreamer.cpp     # COFF 目标文件 Streamer
│   │   ├── MCNullStreamer.cpp      # 空 Streamer
│   │   └── ...
│   │
│   ├── CodeGen/
│   │   └── AsmPrinter/             # ✅ AsmPrinter 基类通用实现
│   │       ├── AsmPrinter.cpp
│   │       ├── DwarfDebug.cpp
│   │       └── ...
│   │
│   └── Target/
│       └── X86/                    # ✅ X86 目标目录
│           ├── X86AsmPrinter.cpp   # X86 的 AsmPrinter 子类
│           ├── X86AsmPrinter.h
│           └── MCTargetDesc/       # ✅ X86 的 MC 目标描述子目录
│               ├── X86MCCodeEmitter.cpp   # X86 的 MCCodeEmitter 子类
│               ├── X86MCCodeEmitter.h
│               ├── X86MCAsmInfo.cpp       # X86 汇编信息
│               ├── X86MCAsmBackend.cpp    # X86 汇编后端
│               ├── X86ELFStreamer.cpp     # （如需）X86 特有的 ELF Streamer
│               └── ...
```



