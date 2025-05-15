---
title: 逻辑推理之形式逻辑
date: 2025-02-27
mathjax: true
categories: 哲学
tags:
- 逻辑
- 哲学
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

这里面只有演绎推理和完全归纳是必然的严密的推理。

演绎推理需要我们严格遵循演绎逻辑，比如三段论中必须遵循主体内涵和外延一致，否则极容易出现“偷换概念”、“红鲱鱼”等谬误。

归纳推理中完全归纳是严密的推理形式，但是真实世界有太多的东西是我们没法通过枚举的形式进行完全归纳的，所以必然要用到不完全归纳。想要保证不完全归纳的结论可靠，就需要用科学归纳法，否则容易产生“以偏概全”谬误。如“我遇到的三个东北人都豪爽→东北人豪爽”，“我发现三个江西人都喜欢吃辣→江西人都喜欢吃辣”。这是人类“经验主义”惯性思维方式，更节省精力，符合进化形成的生存策略。传统文化中有许多这样的“经验主义”的“智慧”，这种思维惯性需要警惕。

类比推理从它的英文“Analogical”词根“ana”可以看出它是不符合逻辑的，表示“按比例对应”。类比只适用于让人快速理解某个概念，类比推理方式本身不是严密的逻辑推理。中国论证中频繁使用类比的现象，背后有着深刻的文化、思维和社会根源，中国有隐喻文化，社会语境中一直倡导委婉表达的智慧。比如《孟子》的“鱼，我所欲也；熊掌，亦我所欲也。二者不可得兼，舍鱼而取熊掌者也。生，亦我所欲也；义，亦我所欲也。二者不可得兼，舍生而取义者也。”用鱼和熊掌来类比“舍生取义”；《荀子·劝学》中大量的类比论证，“蚓无爪牙之利，筋骨之强，上食埃土，下饮黄泉，用心一也。蟹六跪而二螯，非蛇鳝之穴无可寄托者，用心躁也”。中国的类比传统，既是文化基因的延续，也是适应社会需求的沟通策略。它并非替代逻辑论证，而是作为补充手段，在降低认知门槛、增强说服感染力方面发挥着独特作用。

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

![逆命题与否命题](https://p3-sdbk2-media.byteimg.com/tos-cn-i-xv4ileqgde/af8239e48b3e406dab32c34757b806aa~tplv-xv4ileqgde-image.image)

## 三段论的格式

![三段论的格式](https://bkimg.cdn.bcebos.com/pic/902397dda144ad348afa2597d3a20cf431ad855d)

## 论证错误

```plantuml
@startmindmap
<style>
node {
    HorizontalAlignment center
    RoundCorner 20
}
</style>
* 常\n见\n论\n证\n谬\n误
**_ 论点错误
*** [[https://baike.baidu.com/item/偷换概念/6714008 偷换概念]]
*** 绝对化表述
**_ 论据错误
*** 论据不充分
*** 论据不相干
**** [[https://baike.baidu.com/item/诉诸权威 诉诸权威]]\n(迷信权威)
**** [[https://baike.baidu.com/item/诉诸大众 诉诸众人]]\n(盲目从众)
**** [[https://baike.baidu.com/item/诉诸无知 诉诸无知]]\n(以无知为据)
**** [[https://baike.baidu.com/item/诉诸情感 诉诸情感]]\n(情感绑架)
*****_ 诉诸怜悯
*****_ 诉诸仇恨
*****_ 诉诸恐惧
*****_ 诉诸厌恶
**** 诉诸人身\n([[https://baike.baidu.com/item/人身攻击 人身攻击]])
*****_ 诉诸动机
*****_ 诉诸人格
*** [[https://baike.baidu.com/item/预期理由 预期理由]]\n(不当预设)
**_ 论证错误
*** 归纳不当
**** [[https://baike.baidu.com/item/以偏概全 以偏概全]]
**** [[https://baike.baidu.com/item/统计陷阱/3778566 统计学谬误]]
*****_ 数据偏差
******_ [[https://baike.baidu.com/item/幸存者偏差/10313799 幸存者偏差]]\n(样本选择偏差)
******_ [[https://baike.baidu.com/item/观察者偏差 观察者偏差]]\n(无意识的双重标准)
*****_ 数据分析偏差
******_ [[https://baike.baidu.com/item/区群谬误 区群谬误]]\n(以全概偏)
******_ [[https://baike.baidu.com/item/合成谬误 合成谬误]]\n(以偏概全)
*** 类比不当
*** 因果不当
**** 强拉因果\n([[https://baike.baidu.com/item/基本归因错误 错误归因]])
**** [[https://baike.baidu.com/item/因果倒置 因果倒置]]
**** 因果矛盾
**** [[https://baike.baidu.com/item/滑坡谬误 滑坡谬误]]
*** 非黑即白
@endmindmap
```

> [謬誤列表](https://zh.wikipedia.org/wiki/謬誤列表)
