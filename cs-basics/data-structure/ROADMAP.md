# 数据结构（408）路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 7 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([408 数据结构 45 分]) --> M1[模块一: 地基]
    M1 --> DS01[DS01 绪论与复杂度]

    DS01 --> M2[模块二: 线性结构]
    M2 --> DS02[DS02 线性表]
    M2 --> DS03[DS03 栈队列与数组]
    M2 --> DS04[DS04 串与 KMP]

    DS02 --> M3[模块三: 非线性结构]
    M3 --> DS05[DS05 树与二叉树]
    M3 --> DS06[DS06 图]

    DS05 --> M4[模块四: 查找与排序]
    M4 --> DS07[DS07 查找]
    M4 --> DS08[DS08 排序]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style DS05 fill:#ffcdd2
    style DS06 fill:#ffcdd2
    style DS07 fill:#ffcdd2
```

> 红色 = 大题高频命中区（树 / 图 / 查找）。

---

## 🧭 数据结构全景分类

```mermaid
graph TB
    Data[数据结构] --> Logic[逻辑结构]
    Data --> Store[存储结构]

    Logic --> Linear[线性结构]
    Logic --> NonLinear[非线性结构]

    Linear --> L1[线性表 DS02]
    Linear --> L2[栈 队列 DS03]
    Linear --> L3[串 DS04]

    NonLinear --> N1[树 DS05]
    NonLinear --> N2[图 DS06]
    NonLinear --> N3[集合 查找表 DS07]

    Store --> S1[顺序存储<br>随机访问 O1]
    Store --> S2[链式存储<br>插删不移动]
    Store --> S3[索引存储]
    Store --> S4[散列存储 DS07]

    style Linear fill:#bbdefb
    style NonLinear fill:#fff9c4
    style Store fill:#c8e6c9
```

---

## 📈 复杂度分析决策树（DS01）

```mermaid
flowchart TD
    Q{程序形态}
    Q -->|非递归| A[数循环层数<br>写求和式 化简]
    Q -->|递归| B{递推式形态}
    B -->|T n = aT n/b + f n| C[主定理三种情形]
    B -->|减而治之 T n = T n-1 + f n| D[递归树/展开法]
    C --> C1[f n 小于 n^logba → O n^logba]
    C --> C2[f n 等于 n^logba → 乘 log n]
    C --> C3[f n 大于 n^logba → O f n]

    Q -->|带扩容/摊还| E[聚合法 或 势能法]

    style C fill:#ffcdd2
    style E fill:#fff9c4
```

---

## 🔗 线性表选型（DS02）

```mermaid
flowchart LR
    Q{主要操作是什么}
    Q -->|按下标随机访问多| Seq[顺序表<br>O1 访问 On 插删]
    Q -->|频繁在已知位置插删| List[链表<br>On 访问 O1 插删]
    Q -->|需要往前找前驱| DList[双链表]
    Q -->|需要循环遍历/约瑟夫| CList[循环链表]
    Q -->|无指针语言/408 考纲| SList[静态链表<br>游标模拟]

    style Seq fill:#c8e6c9
    style List fill:#bbdefb
    style SList fill:#eeeeee
```

**易错点**：「链表插入 O(1)」的前提是**已经拿到前驱指针**；按位置插入仍要 O(n) 找位置。

---

## 🔄 循环队列三种判满方案（DS03 最高频）

```mermaid
graph TB
    subgraph 方案一 牺牲一个存储单元
        A1["队空: front == rear"]
        A2["队满: (rear+1)%MaxSize == front"]
        A3["元素数: (rear - front + MaxSize) % MaxSize"]
    end
    subgraph 方案二 增设 size 计数器
        B1["队空: size == 0"]
        B2["队满: size == MaxSize"]
        B3["可用满 MaxSize 个格子"]
    end
    subgraph 方案三 增设 tag 标志
        C1["tag=0 表示上次是删除 → front==rear 即空"]
        C2["tag=1 表示上次是插入 → front==rear 即满"]
    end
    style A2 fill:#ffcdd2
    style B2 fill:#c8e6c9
    style C2 fill:#fff9c4
```

---

## 🌲 二叉树遍历与还原（DS05）

```mermaid
flowchart TD
    T{已知哪两个序列}
    T -->|前序 + 中序| Y1[✅ 唯一确定]
    T -->|后序 + 中序| Y2[✅ 唯一确定]
    T -->|层序 + 中序| Y3[✅ 唯一确定]
    T -->|前序 + 后序| N1[❌ 不唯一<br>无法区分左右单分支]
    T -->|前序 + 层序| N2[❌ 不唯一]

    Y1 --> Key[关键: 中序提供左右子树的分界]
    Y2 --> Key
    Y3 --> Key

    style Y1 fill:#c8e6c9
    style N1 fill:#ffcdd2
    style Key fill:#fff9c4
```

---

## 🕸️ 图算法选型（DS06）

```mermaid
flowchart TD
    Q{问题类型}
    Q -->|最小生成树| MST{图的稠密度}
    MST -->|稠密图 边多| Prim["Prim O V²<br>从点出发"]
    MST -->|稀疏图 边少| Kruskal["Kruskal O E log E<br>从边出发 + 并查集"]

    Q -->|单源最短路 非负权| Dij["Dijkstra O V²"]
    Q -->|单源最短路 有负权| Bell[Bellman-Ford]
    Q -->|多源最短路| Floyd["Floyd O V³<br>k 必须在最外层"]
    Q -->|无权图最短路| BFS[BFS 层数即距离]

    Q -->|有向图判环/排序| Topo[拓扑排序<br>入度为 0 入队]
    Q -->|工程最短工期| CPM[AOE 关键路径<br>ve/vl/e/l]

    style Dij fill:#ffcdd2
    style Floyd fill:#ffcdd2
    style Topo fill:#c8e6c9
```

---

## 🔍 查找结构对比（DS07）

```mermaid
graph TB
    subgraph 静态查找
        Seq["顺序查找 ASL=(n+1)/2"]
        Bin["折半查找 O log n<br>需有序+顺序存储"]
        Blk[分块查找<br>块间有序块内无序]
    end
    subgraph 动态查找树
        BST[BST<br>最坏退化成链 On]
        AVL[AVL<br>严格平衡 查找快 旋转多]
        RB[红黑树<br>近似平衡 插删快<br>map/TreeMap]
        B[B 树<br>多路平衡 磁盘友好]
        BP[B+ 树<br>数据全在叶 链表串联<br>数据库索引]
    end
    subgraph 散列
        Hash["散列表 平均 O1<br>装填因子 α 决定 ASL"]
    end

    style AVL fill:#ffcdd2
    style BP fill:#c8e6c9
    style Hash fill:#fff9c4
```

---

## 🔃 AVL 四种旋转（DS07 必考画图题）

```mermaid
flowchart LR
    A{失衡结点 A<br>看不平衡出现在哪}
    A -->|左子树的左边 LL| LL[右单旋<br>以 A 的左孩子为轴]
    A -->|右子树的右边 RR| RR[左单旋<br>以 A 的右孩子为轴]
    A -->|左子树的右边 LR| LR[先左旋后右旋<br>提升孙子结点]
    A -->|右子树的左边 RL| RL[先右旋后左旋<br>提升孙子结点]

    style LL fill:#c8e6c9
    style RR fill:#c8e6c9
    style LR fill:#ffcdd2
    style RL fill:#ffcdd2
```

**口诀**：单旋提儿子，双旋提孙子。

---

## 📊 八大排序全景（DS08）

```mermaid
graph TB
    Sort[内部排序] --> Ins[插入类]
    Sort --> Swap[交换类]
    Sort --> Sel[选择类]
    Sort --> Mer[归并类]
    Sort --> Rad[基数类]

    Ins --> I1["直接插入 On² 稳定"]
    Ins --> I2["折半插入 比较 O n log n 移动 On²"]
    Ins --> I3["希尔 增量序列 不稳定"]

    Swap --> W1["冒泡 On² 稳定"]
    Swap --> W2["快排 平均 O n log n 不稳定<br>最坏 On² 递归栈 O log n"]

    Sel --> E1["简单选择 On² 不稳定"]
    Sel --> E2["堆排 O n log n 不稳定<br>建堆 On"]

    Mer --> M1["归并 O n log n 稳定 空间 On"]
    Rad --> R1["基数 O d n+r 稳定"]

    style W2 fill:#ffcdd2
    style E2 fill:#ffcdd2
    style M1 fill:#c8e6c9
```

---

## 🎯 排序算法选型

```mermaid
flowchart TD
    Q{约束条件}
    Q -->|n 很小 或 基本有序| A[直接插入排序]
    Q -->|要求稳定 + O n log n| B[归并排序]
    Q -->|要求原地 + 平均最快| C[快速排序]
    Q -->|要求最坏也是 O n log n + 原地| D[堆排序]
    Q -->|只求前 K 个| E[堆 / 快速选择]
    Q -->|整数且范围小| F[计数 / 基数排序]
    Q -->|数据放不进内存| G[外部排序<br>归并段 + 败者树]

    style C fill:#ffcdd2
    style B fill:#c8e6c9
    style G fill:#fff9c4
```

---

## 🎓 建议学习顺序

```mermaid
graph LR
    A[DS01<br>复杂度] --> B[DS02-DS04<br>线性结构]
    B --> C[DS05-DS06<br>树与图]
    C --> D[DS07-DS08<br>查找与排序]
    D --> E[真题 + 错题复盘]

    style A fill:#c8e6c9
    style C fill:#ffcdd2
    style E fill:#e1bee7
```

按顺序读 = 完整 408 数据结构能力；面试突击见 [INDEX.md](./INDEX.md) 的「路径 B」。
