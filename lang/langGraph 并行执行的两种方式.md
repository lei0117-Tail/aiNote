

#并行
#langgraph


### 方式一：静态并行 —— `add_edge(list, node)`（扇入收敛）

**一个节点 → 多个并行节点 → 一个收敛节点**

原理：

- **扇出（fan-out）**：用多条 `add_edge` 从同一个节点指向多个目标，LangGraph 会自动并行执行
- **扇入（fan-in）**：`add_edge(["节点A", "节点B"], "收敛节点")` 传入**列表**，表示等待所有节点完成后再继续

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAwCAYAAABXAvmHAAADO0lEQVR4Xu2Zz0rjUBTGq8KsnMUs3IhPMEvfQxc2AbHpYuYdFMRNVw6p8wR9AAWfoVDQ1hbE5SwKVUFcdTaa2iZC2zvnlMS5Ofc0fzpNGph+8CMQb7/zpbm596TmckstlQEJsaKbzq5m2hWtbLfyZfsRjiNAuIzccy0co5sDTS+JT9RmIdJ/vn/Nm3ZbChsJuKAOfpb6paqDHy9fIEyXhotBFz2ob2rSz+yTjzCmfQzTYh2PTNCpoAf1TU0QoOoF2TXFZzyHRxoyhCr1TU0wj58+gsx4B9CD+qYmCPBGA83AG/VNTVB8yASKy5D6piYoPmYCxWVMfTMh3KiiPg/0s5lR1BWJfi4TWugdwN3R3aSqbg/j0KLzRK5tNQzh4liNwuNrw6j2GoWTl+uDaDs29ifav7UHsZHrSxdA6fauCsG9E952bLJogaSRMzDBZTrilz69i82XB3lqngZyBia0j17DyMvjfXL7eaVAwjhefXGpr9HAKsWKnNknvWw3mQJJc+vV79f3N9XACk05s09g9swUSJjBkVffui7uMIEpz3Jmn8CwrxZIlJp+Kda8+jC/z5nAPmBMX87skzaf5iyMd73s3MF78aEc/qVpbEPAEQ3MML350+JfwAge/Iv8mbOzZw62qF9U9a6+bVg3xgMTliPwAuJMofu9cn+besSV+83fM0FZwqZQ1If4QT/tbdDPR9WgVdjCBxbahAsr2rSRmf4QR1xGR/I3P1m768YhfDN3YP7OFJw3ActotI3s3Bs/CX9j1JgiCRKwkeHDyAT2gWO88WB2pBZIGJh6cmafSiWxGvYr277Z3/TGg+GtUiBZ2kKUVuXMisLaaXnttiY9u1IkKcLbaU9BLzTyOKbIPJnthSaOmKIq9eJxYA+/SClhGX7Xv09+csykaFiWjN+BsRI4Pov7YQuKD5lAcZnenCUt2IXfmEDxAA/qm5peG8UnJVAQ8Dx0a/o6Hr1z6EF9UxOu0UrIALwVCY9/L8BY3D84cIOhIQNh7gB6UN/UhLsjhOgqQaPTTWSHjSPsTyBImwkXRidyb5O0hMitwEvNLvbqsKq0sIex/G9ao8m5yd+KFZg2WmY3t6X+N/0Bnquc7URUP5cAAAAASUVORK5CYII=)

py

Apply
~~~python
from langgraph.graph import StateGraph, END, START
import operator
from typing import Annotated, TypedDict, List

class State(TypedDict):
    messages: Annotated[List[str], operator.add]  # 追加模式，并行结果自动合并

def node_start(state): 
    return {"messages": ["start完成"]}

def node_a(state):
    return {"messages": ["A完成"]}   # 并行分支1

def node_b(state):
    return {"messages": ["B完成"]}   # 并行分支2

def node_merge(state):
    # 此时 state["messages"] 已包含 A 和 B 的结果（顺序不确定）
    return {"messages": [f"merge收到: {state['messages']}"]}

workflow = StateGraph(State)
workflow.add_node("start", node_start)
workflow.add_node("branch_a", node_a)
workflow.add_node("branch_b", node_b)
workflow.add_node("merge", node_merge)

workflow.set_entry_point("start")

# 扇出：start → 同时触发 A 和 B
workflow.add_edge("start", "branch_a")
workflow.add_edge("start", "branch_b")

# 扇入：等 A 和 B 都完成，再执行 merge
workflow.add_edge(["branch_a", "branch_b"], "merge")   # ← 关键！传列表

workflow.add_edge("merge", END)
app = workflow.compile()
~~~

~~~plain text
start → branch_a ┐
                  ├→ merge → END
       branch_b ┘
(A和B并行，merge等两者都完成)
~~~

### 方式二：动态并行（Map-Reduce）—— `Send` API

**适合数量不固定的并行任务**，比如对一个列表里每个元素并行处理。
~~~python
from langgraph.types import Send
from langgraph.graph import StateGraph, END, START
import operator
from typing import Annotated, TypedDict, List

class OverallState(TypedDict):
    tasks: List[str]                              # 待处理的任务列表
    results: Annotated[List[str], operator.add]   # 收集各并行结果（追加模式）

class WorkerState(TypedDict):   # 每个 worker 子任务的独立状态
    task: str

def dispatch_node(state: OverallState):
    # 扇出：对 tasks 里每个任务，各自启动一个 worker
    # 每个 Send 对应一个独立的并行执行
    return [Send("worker", {"task": t}) for t in state["tasks"]]

def worker_node(state: WorkerState):
    # 每个 worker 独立处理自己的 task
    result = f"处理结果: {state['task'].upper()}"
    return {"results": [result]}   # 结果追加回 OverallState

def merge_node(state: OverallState):
    # 所有 worker 完成后收敛到这里
    print("所有并行结果:", state["results"])
    return {}

workflow = StateGraph(OverallState)
workflow.add_node("worker", worker_node)
workflow.add_node("merge", merge_node)

# dispatch 逻辑直接写在 conditional_edges 里
workflow.add_conditional_edges(START, dispatch_node)   # ← 返回 [Send(...), Send(...)]
workflow.add_edge("worker", "merge")
workflow.add_edge("merge", END)

app = workflow.compile()
app.invoke({"tasks": ["任务1", "任务2", "任务3"], "results": []})
~~~



~~~plain text

START → dispatch → worker("任务1") ┐
                 → worker("任务2") ├→ merge → END
                 → worker("任务3") ┘
(3个worker完全并行，merge等所有完成)
~~~
---

### 方式三：并行子图（Subgraph）

~~~python
# 子图1
sub1 = StateGraph(SubState)
sub1.add_node("sub1_step1", ...)
sub1.add_node("sub1_step2", ...)
sub1.add_edge("sub1_step1", "sub1_step2")
sub1.set_entry_point("sub1_step1")
sub1.set_finish_point("sub1_step2")
compiled_sub1 = sub1.compile()

# 子图2
sub2 = StateGraph(SubState)
...
compiled_sub2 = sub2.compile()

# 主图：把编译好的子图作为节点添加进来
main = StateGraph(MainState)
main.add_node("start", start_node)
main.add_node("sub1", compiled_sub1)   # ← 子图直接作为节点
main.add_node("sub2", compiled_sub2)
main.add_node("merge", merge_node)

main.set_entry_point("start")
main.add_edge("start", "sub1")    # 扇出
main.add_edge("start", "sub2")
main.add_edge(["sub1", "sub2"], "merge")   # 扇入收敛
main.add_edge("merge", END)

app = main.compile()
~~~
---

### 三种方式对比

|方式|适用场景|并行数量|状态共享|
|---|---|---|---|
|`add_edge(list, node)`|固定数量的并行节点|固定|共享主图 State|
|`Send` API|动态数量（Map-Reduce）|动态|每个 worker 可有独立 State|
|并行子图|每个分支内部逻辑复杂|固定|子图有自己的 State|

**关键点**：并行结果的收集必须用 `Annotated[List, operator.add]` 追加模式，否则并行写入会互相覆盖！