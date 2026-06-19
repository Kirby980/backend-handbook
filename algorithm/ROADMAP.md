# 算法面试 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始算法进阶]) --> M1[模块1: 基础与方法论]
    M1 --> A01[01 复杂度+方法论]
    M1 --> A02[02 数组字符串]
    M1 --> A03[03 链表]

    A03 --> M2[模块2: 核心题型模板]
    M2 --> A04[04 双指针]
    M2 --> A05[05 滑动窗口]
    M2 --> A06[06 二分查找]
    M2 --> A07[07 排序]
    M2 --> A08[08 回溯DFS]
    M2 --> A09[09 分治]

    M2 --> M3[模块3: 栈队列树图]
    M3 --> A10[10 栈与队列]
    M3 --> A11[11 哈希+设计]
    M3 --> A12[12 二叉树]
    M3 --> A13[13 堆 TopK]
    M3 --> A14[14 图论]

    M3 --> M4[模块4: DP与贪心]
    M4 --> A15[15 DP 基础]
    M4 --> A16[16 经典 DP]
    M4 --> A17[17 贪心]

    M4 --> M5[模块5: 进阶与总结]
    M5 --> A18[18 字符串高级]
    M5 --> A19[19 位运算数学]
    M5 --> A20[20 总结+刷题路线]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#e1bee7
    style A06 fill:#ffcdd2
    style A08 fill:#ffcdd2
    style A12 fill:#ffcdd2
    style A15 fill:#ffcdd2
    style A16 fill:#ffcdd2
```

---

## 🧭 解题方法论五步法

```mermaid
flowchart LR
    R[1 澄清<br>输入/输出/边界/规模] --> E[2 暴力解<br>先能跑]
    E --> O[3 优化<br>识别题型套模板]
    O --> C[4 编码<br>+测边界]
    C --> D[5 复杂度与权衡]
    style R fill:#c8e6c9
    style O fill:#fff9c4
    style D fill:#ffccbc
```

---

## 🌲 题型 → 模板 速查决策树

```mermaid
flowchart TD
    Q{看到题先问:<br>什么数据结构/什么目标}
    Q -->|有序数组/找边界/最值判定| BS[二分查找]
    Q -->|连续子数组/子串 + 最长最短| SW[滑动窗口]
    Q -->|两端/有序找配对| TP[双指针]
    Q -->|所有方案/排列组合子集| BT[回溯 DFS]
    Q -->|树形结构| Tree[二叉树递归]
    Q -->|最短路径/层级/连通| Graph[BFS/DFS/并查集]
    Q -->|前K大小/动态最值| Heap[堆/优先队列]
    Q -->|下一个更大/区间最值| Stack[单调栈/单调队列]
    Q -->|最优解/计数/能否到达 + 重叠子问题| DP[动态规划]
    Q -->|局部最优能推全局| Greedy[贪心]

    style BS fill:#ffcdd2
    style DP fill:#ffcdd2
    style SW fill:#fff9c4
    style BT fill:#c8e6c9
```

---

## 🔁 回溯通用模板（08）

```mermaid
flowchart TD
    Start[backtrack 路径, 选择列表] --> Check{满足结束条件?}
    Check -->|是| Add[记录结果, return]
    Check -->|否| Loop[遍历选择列表]
    Loop --> Prune{剪枝: 不合法?}
    Prune -->|是| Loop
    Prune -->|否| Make[做选择 加入路径]
    Make --> Recurse[递归 backtrack]
    Recurse --> Undo[撤销选择 回溯]
    Undo --> Loop
    style Make fill:#c8e6c9
    style Undo fill:#ffccbc
    style Prune fill:#fff9c4
```

---

## 🧮 动态规划四步法（15）

```mermaid
flowchart LR
    S1[1 定义状态<br>dp i 表示什么] --> S2[2 转移方程<br>dp i 由谁推来]
    S2 --> S3[3 初始化<br>边界 base case]
    S3 --> S4[4 遍历顺序<br>保证依赖先算]
    S4 --> S5[5 优化<br>滚动数组降维]
    style S1 fill:#c8e6c9
    style S2 fill:#fff9c4
    style S5 fill:#bbdefb
```

---

## 📊 排序算法对比（07）

```mermaid
graph TB
    subgraph 比较排序 O n log n
        Quick[快速排序<br>原地·不稳定·平均最快]
        Merge[归并排序<br>稳定·需O n 空间·链表友好]
        Heap[堆排序<br>原地·不稳定·TopK]
    end
    subgraph 非比较排序 O n
        Count[计数排序<br>范围小整数]
        Bucket[桶排序]
        Radix[基数排序]
    end
    style Quick fill:#ffcdd2
    style Merge fill:#c8e6c9
    style Count fill:#fff9c4
```

---

## 🎓 学习顺序建议

```mermaid
graph LR
    A[01-03<br>基础方法论] --> B[04-09<br>核心模板]
    B --> C[10-14<br>栈队列树图]
    C --> D[15-17<br>DP与贪心]
    D --> E[18-20<br>进阶+总结]

    style A fill:#c8e6c9
    style B fill:#bbdefb
    style D fill:#ffccbc
```

按顺序读 = 完整算法面试能力；面试突击 = 看 [INDEX.md](./INDEX.md) 的「路径 A」。
