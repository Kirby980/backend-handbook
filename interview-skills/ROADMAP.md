# 求职与面试软技能 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月**

---

## 🗺️ 求职全流程

```mermaid
graph TD
    Start([决定求职]) --> M1[模块1: 简历与定位]
    M1 --> I01[I01 简历写作]
    M1 --> I02[I02 项目怎么写]
    M1 --> I03[I03 JD 对标]

    I03 --> M2[模块2: 面试表达]
    M2 --> I04[I04 自我介绍]
    M2 --> I05[I05 项目深挖]
    M2 --> I06[I06 行为面试]

    M2 --> M3[模块3: 高频环节与收尾]
    M3 --> I07[I07 HR 问题]
    M3 --> I08[I08 反问环节]
    M3 --> I09[I09 薪资谈判]
    M3 --> I10[I10 表达与复盘]

    I10 -.复盘后下一场.-> M2

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#ffccbc
    style I05 fill:#ffcdd2
```

---

## 📄 简历的"会做→看得见"翻译（I01-I02）

```mermaid
flowchart LR
    Did[我做了 X] --> Q1{招聘方关心啥?}
    Q1 --> Imp[影响/结果<br>用数字量化]
    Q1 --> How[怎么做的<br>技术/难点/角色]
    Imp --> STAR[STAR 一句话:<br>情境-任务-行动-结果]
    How --> STAR
    STAR --> Defend{经得起追问?}
    Defend -->|否| Cut[删/改，别写不敢被问的]
    Defend -->|是| Keep[保留，并预判追问准备答案]

    style STAR fill:#fff9c4
    style Defend fill:#ffcdd2
```

---

## 🔥 项目深挖应对（I05·面试最硬一关）

```mermaid
flowchart TD
    Q[面试官: 讲讲你的项目] --> Overview[1 一句话定位:<br>做什么/规模/你的角色]
    Overview --> Arch[2 架构: 整体设计 + 数据流]
    Arch --> Probe{开始追问}
    Probe --> Why[为什么这么设计?<br>讲取舍/备选方案]
    Probe --> Hard[最难的点?<br>讲难点+如何解决]
    Probe --> Scale[扛多少量?<br>容量/瓶颈/优化]
    Probe --> Fail[出过什么问题?<br>线上故障+复盘]
    Why --> Honest{答不上来?}
    Hard --> Honest
    Scale --> Honest
    Fail --> Honest
    Honest -->|诚实| Say[坦诚说边界 + 讲思路<br>胜过硬编]
    Honest -->|硬编| Trap[❌ 一追就穿，扣分更狠]

    style Overview fill:#c8e6c9
    style Probe fill:#fff9c4
    style Say fill:#bbdefb
    style Trap fill:#ffcdd2
```

---

## 🎭 STAR 法则（I06 行为面试）

```mermaid
flowchart LR
    S[Situation 情境<br>背景，简短] --> T[Task 任务<br>你的目标/职责]
    T --> A[Action 行动<br>你具体做了什么<br>★占比最大]
    A --> R[Result 结果<br>量化 + 你的收获]
    style A fill:#fff9c4
    style R fill:#c8e6c9
```

> 重点在 **A(你做了什么)**——很多人花太多时间讲 S(背景),结果听不出"你"的贡献。

---

## 💰 谈薪决策（I09）

```mermaid
flowchart TD
    Ask[HR: 期望薪资?] --> Stage{在什么阶段?}
    Stage -->|刚开始/没拿offer| Defer[先了解范围/给区间<br>别过早报死数]
    Stage -->|已有 offer| Anchor[基于行情 + 自身价值锚定]
    Anchor --> Multi{多个 offer?}
    Multi -->|是| Compare[综合比较:<br>钱/成长/团队/稳定/方向]
    Multi -->|否| Negotiate[礼貌争取，给理由]
    Compare --> Decide[理性决策，非只看钱]

    style Anchor fill:#fff9c4
    style Decide fill:#c8e6c9
```

---

## 🎓 学习顺序

```mermaid
graph LR
    A[I01-I03<br>简历与定位] --> B[I04-I06<br>面试表达]
    B --> C[I07-I10<br>环节与收尾]
    C -.每场复盘.-> B
    style A fill:#c8e6c9
    style B fill:#bbdefb
    style C fill:#ffccbc
```

投递前看模块一,面试前主攻模块二 + I10,offer 阶段看 I09。
