---
title: 逻辑推理之形式逻辑
date: 2025-02-27
tags:
    - 逻辑
    - 哲学
categories: 哲学
---

形式逻辑可以追溯到古希腊的亚里士多德时期，他在《工具论》中系统阐述三段论，因此被称为“形式逻辑之父”。后来的斯多葛学派又提出逻辑连接词（如“且”“或”“非”）的形式分析，充实了形式逻辑。

在中国则主要是名家与墨家（战国时期），墨家提出“辩”的概念，在《[墨辩](https://baike.baidu.com/item/墨辩)》中探讨“名实关系”与逻辑悖论（如“白马非马”）；名家惠施、公孙龙关注概念分析与语言逻辑。

近代文艺复兴后，莱布尼茨首次用代数符号表达逻辑关系，因此他也被视为“数理逻辑”先驱。19世纪布尔创立“布尔代数”为计算机逻辑电路奠定数学基础。二战时期，图灵提出“图灵机”模型，将逻辑与计算理论结合，直接影响计算机科学。

## 推理的分类

```plantuml
@startmindmap
<style>
node {
    HorizontalAlignment center
    RoundCorner 40
}
</style>
* 推理
**[#LightGreen] 演绎推理(Deductive)\n一般到特殊\n原因推结果
** 归纳推理(Inductive)\n特殊到一般
***[#LightGreen] 完全归纳
***[#Orange] 不完全归纳
**[#Orange] 类比推理(Analogical)\n特殊到特殊
**[#Orange] 溯因推理(Abductive)\n结果推原因
legend left
  <font color="green">绿色 必然性推理</font>
  <font color="orange">橙色 可能性推理</font>
endlegend
@endmindmap
```

## 命题的种类

```plantuml
@startmindmap
* 命题
** 简单命题
*** [[#categorical-proposition 直言命题]]
****_ 全称命题
*****_ <latex>\forall x \in M</latex>
****_ 特称命题
*****_ <latex>\exists x \in M</latex>
****_ 单称命题
*****_ <latex>X \in M</latex>
*** [[#modal-proposition 模态命题]]
****_ 必然性命题
*****_ <latex>\Box x \in M</latex>
****_ 事实性命题
*****_ <latex>X \in M</latex>
****_ 可能性命题
*****_ <latex>\Diamond x \in M</latex>
*** [[#relational-proposition 关系命题]]
****_ 传递性关系命题
****_ 对称性关系命题
** 复合命题
*** 负命题(非)
****_ <latex>\neg x</latex>
*** [[#and-or 联言命题(与)]]
****_ <latex>x \land y</latex>
*** [[#and-or 选言命题(或)]]
****_ 相容选言命题
*****_ <latex>x \lor y</latex>
****_ 不相容选言命题(异或)
*****_ <latex>\bar{x \lor y}</latex>
*** [[#if-then 假言命题]]
****_ 充分条件假言命题
*****_ <latex>x \to y</latex>
****_ 必要条件假言命题
*****_ <latex>x \gets y</latex>
****_ 充要条件假言命题
*****_ <latex>x \leftrightarrow y</latex>
@endmindmap
```

## 对当关系

**直言命题的对当方阵**

```plantuml
@startuml
skinparam componentStyle rectangle

[所有A都是B] . [所有A都不是B]:上反对
[所有A都是B] -- (矛盾)
(矛盾) -- [有的A不是B]
[所有A都不是B] -- (矛盾)
(矛盾) -- [有的A是B]
[有的A是B] . [有的A不是B]:下反对
[所有A都是B] --> [某个a是B]:从属
[某个a是B] --> [有的A是B]:从属
[所有A都不是B] --> [某个a不是B]:从属
[某个a不是B] --> [有的A不是B]:从属
[某个a是B] --right-- (矛盾)
(矛盾) --right-- [某个a不是B]

[某个a是B]-[hidden]-(a)
[某个a不是B]-[hidden]-(b)
hide a
hide b
@enduml
```

**模态命题的对当方阵**

```plantuml
@startuml
skinparam componentStyle rectangle

[A必然是B] . [A必然不是B]:上反对
[A必然是B] -- (矛盾)
(矛盾) -- [A可能不是B]
[A必然不是B] -- (矛盾)
(矛盾) -- [A可能是B]
[A可能是B] . [A可能不是B]:下反对
[A必然是B] --> [a是B]:从属
[a是B] --> [A可能是B]:从属
[A必然不是B] --> [a不是B]:从属
[a不是B] --> [A可能不是B]:从属
[a是B] --right-- (矛盾)
(矛盾) --right-- [a不是B]

[a是B]-[hidden]-(a)
[a不是B]-[hidden]-(b)
hide a
hide b
@enduml
```

## 逆命题与否命题

```plantuml
@startuml
skinparam componentStyle rectangle
<style>
component{
HorizontalAlignment center
}
</style>

[原命题\n若p，则q] <-right-> [逆命题\n若q，则p]:互逆命题，真假无关
[原命题\n若p，则q] <-- (互为\n逆否)
(互为\n逆否) --> [逆否命题\n若非q，则非p]
[逆命题\n若q，则p] <-- (互为\n逆否)
(互为\n逆否) --> [否命题\n若非p，则非q]
[否命题\n若非p，则非q] <-right-> [逆否命题\n若非q，则非p]:互逆命题，真假无关
[原命题\n若p，则q] <-down-> [否命题\n若非p，则非q]:互否命题\n真假无关
[逆命题\n若q，则p] <-down-> [逆否命题\n若非q，则非p]:互否命题\n真假无关
@enduml
```

## 三段论的格式

![三段论的格式](https://bkimg.cdn.bcebos.com/pic/902397dda144ad348afa2597d3a20cf431ad855d)

