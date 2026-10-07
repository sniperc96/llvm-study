## SDValue和SDNode

SDValue的定义如下：

```c++
//===----------------------------------------------------------------------===//
/// Unlike LLVM values, Selection DAG nodes may return multiple
/// values as the result of a computation. Many nodes return multiple values,
/// from loads (which define a token and a return value) to ADDC (which returns
/// a result and a carry value), to calls (which may return an arbitrary number
/// of values).
///
/// As such, each use of a SelectionDAG computation must indicate
/// the node that computes it as well as which return value to use from that
/// node. This pair of information is represented with the SDValue value type.
///
class SDValue {
  friend struct DenseMapInfo<SDValue>;

  SDNode *Node = nullptr; // The node defining the value we are using.
  unsigned ResNo = 0;     // Which return value of the node we are using.

public:
  SDValue() = default;
  SDValue(SDNode *node, unsigned resno);

  // ... (getter methods, etc.)
};
```

 `SDNode` 可以产生多个结果，每个结果都是一个 `SDValue`，通过 `ResNo`（Result Number）来区分。

这在 LLVM 源码中有明确说明：**“与 LLVM 值不同，SelectionDAG 节点可能作为计算的结果返回多个值”**。例如：

- 一个 `load` 节点通常会产生**两个结果**：一个是加载的数据值（`ResNo = 0`），另一个是用于副作用排序的 `chain` 值（`ResNo = 1`）。
- 一个 `divrem` 节点会同时产生商和余数两个值。

### 例子

对于 `%a = load i32, i32* %p` 这行 IR：

- **`SDNode`**：是那个 `LOAD` 操作节点，它代表了“加载”这个行为。
- **`SDValue`**：`%a` 是一个 `SDValue`，它的 `Node` 指针指向 `LOAD` 节点，`ResNo` 为 `0`（假设 0 号结果是加载的数据值）。
- **`%p`**：它也是另一个 `SDValue`（指向产生 `%p` 的节点），作为 `LOAD` 节点的一个操作数（`Operand`）。

总结一下，`SDValue` 是理解 `SelectionDAG` 的关键：**它是对 `SDNode` 多返回值特性的优雅封装**，让 DAG 中的依赖关系（边）可以被精确地表示为“某个节点的第几个输出被另一个节点使用”。

## SDUse