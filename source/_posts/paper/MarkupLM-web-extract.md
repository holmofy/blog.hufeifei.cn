---
title: 【译】基于MarkupLM的web数据抽取
date: 2023-01-26
categories: 算法
tags: 
- 算法
- Web挖掘
---

## 摘要

网站在当今数字时代已成为许多组织获取信息的关键来源。然而，从多个网站的网页中提取和组织半结构化数据存在挑战，尤其是在希望保持广泛适用性的同时实现高度自动化时。在追求自动化的过程中，自然而然的发展是将网页数据提取的方法从仅能处理单个网站扩展到通常在同一领域内处理多个网站。尽管这些网站共享相同的域，但数据的结构可能差异巨大。一个关键问题是在保持足够准确性的同时，这样的系统能够通用地涵盖大量网站。该论文检查了在多个瑞典保险公司网站上进行的自动化网络数据提取的效率。先前的工作表明，使用包含多个领域网页的已知英语数据集可以取得良好的结果。选择了最先进的模型MarkupLM，并使用监督学习使用两个预训练模型（一个瑞典模型和一个英语模型）在标记的汽车保险客户网络数据的训练集上进行零样本学习。结果显示，这样的模型可以通过利用预训练模型，在源语言为瑞典的情况下，以相对较小的数据集在领域范围内取得良好的准确性。

## 1、介绍

数字时代使互联网成为主要信息来源。互联网上的数据丰富且复杂度增加，同时对更复杂服务的需求也在不断增加。尽管有大量数据可供探索，一个关键挑战是在满足数据质量和有效性要求的前提下，尽可能高效而准确地提取和结构化信息。数据的结构范围从非结构化数据（如文本）到半结构化数据（如超文本标记语言（HTML））再到结构化数据，后者可以采用表格或数据库生成的HTML形式 [1, 2]。
尽管人类可以手动提取这些数据，但自动化这一过程是非常可取的，即最小化人工劳动、错误和干预。存在一些可自动提取信息的网络数据提取方法，但它们的使用高度依赖于泛化和鲁棒性要求。
另一种选择是网站提供 Web 应用程序编程接口（API），使用诸如 RESTful API 或 GraphQL API 等技术。然而，在本论文中不会探讨这种替代方案。
另一种选择是网页的行业标准格式。通过模板对网站进行一些标准化，如[3]所述，但本论文不会关注这种替代方案。

### 1.1、背景

自动化网络数据提取的一个主要问题是系统的灵活性和通用性。根据Sergio Flesca等人的说法，许多系统依赖于包装器，“一组适用于从网站提取信息的提取规则” [4]，这些规则与其训练时紧密耦合的网站的底层文档对象模型（DOM）[5]树结构相关。这使得系统对结构的变化非常敏感，除非进行包装器维护 [6]，同时在未在训练集中的网站上提取正确数据方面效果不佳。任何这类系统的一个极具吸引力的特征是从先前未见过的网站提取数据（即，它应具有泛化能力），并且在满足使用提取数据的应用程序的具体要求的同时保持足够的准确性。尝试在生成和维护这样的系统期间最小化涉及的手动人工劳动会进一步增加问题的复杂性。
问题的一个有趣的限定是将自动化网络数据提取系统的泛化能力缩小到一组具有一些相似之处的网站。其中一种方法是创建一个特定领域的系统，旨在从同一垂直（即领域）内的多个网站中提取相同类型的对象（例如，图书）。这使系统能够充分利用这些网站在信息和结构上潜在共享的相似之处。

### 1.2、问题

这个问题在很大程度上依赖于所需数据的复杂性（例如，结构水平和目标属性数量），以及目标领域网站表示（即，HTML布局）的相似性。另一个方面是网站的语言，这是一个依赖自然语言处理（NLP）从文本中提取语义意义的系统（即，模型）中的因素。在训练数据有限时，预训练表示通常对提高性能至关重要。虽然英语有大量高质量的预训练模型，但瑞典语的数量并不如此之多。问题的一个有趣方面是预训练表示对网页数据提取模型性能的影响。
一个带有监督学习的网络数据提取模型能够从未见过的瑞典保险网站中提取信息的效果如何？

### 1.3、宗旨

该论文旨在探索自动化从同一垂直内的多个网站中提取网络数据的可能性。具体而言，将使用瑞典保险网站的用户网页，其中包含其保险计划的摘要。这将有望为使用当前先进技术（SOTA）模型从瑞典保险网站提取数据的可能性和效率提供一些见解。

### 1.4、目标

该论文旨在确定一个适用的网络数据提取模型，然后在瑞典汽车保险网站上对其进行修改和评估。子目标包括：
- 获取数据集，
- 确定适用的模型，
- 修改模型，以及
- 评估模型。

### 1.5、研究方法

项目中采用的研究方法将是设计科学 [7]，并使用实验方法进行评估。设计科学是一种范式，其中通过设计的工件产生知识和解决方案。
该论文将采用 MarkupLM 模型（参见第2.4.5节），并进行必要的修改以使其与瑞典语兼容。该模型（即，工件）将通过实验评估，以确定它在测试数据集中从未见过的保险网站中提取数据的效果如何。准确性将使用三个指标进行测量：精确度、召回率和 F-分数（这些指标在第2.3节中描述）。

### 1.6、限制

该论文探讨并评估单一模型的变种，而非多个不同模型。所使用的数据将仅为瑞典语且为HTML格式。对于数据集的基准真实性，将不进行手动标注。相反，将使用公司（即，Insurely）开发的手工提取机制生成基准真实性。

### 1.7、结构

第二章介绍了有关自动化网络数据提取的相关背景信息。第三章介绍了解决问题所使用的方法和方法论。第四章描述了对先进技术（SOTA）模型的修改。第五章呈现了对模型进行评估的结果。第六章讨论了所获得的结果，最后第七章提出了论文的结论并提出未来的工作。

## 2、背景

这一章概述了与网页数据提取领域（第2.1节）和深度学习（第2.2节）相关的技术，这些技术可能在网页数据提取系统中使用。第2.3节描述了用于评估网页数据提取系统的一些性能指标。不同的网页数据提取方法和三个先进技术（SOTA）模型作为相关工作被介绍（第2.4节）。

### 2.1 网页数据提取
网页数据提取是指从网页中提取信息的过程。软件系统通过在内容更改时自动和重复地从网页中提取数据来执行网页数据提取 [8]。每个网页将如第2.1.1节所述表示，页面的特定部分将如第2.1.2节所述被处理。

#### 2.1.1 文档对象模型

文档对象模型（DOM）是一个API，使得文档（如HTML和可扩展标记语言（XML）文档）能够被表示为逻辑树（如图2.1所示），由节点组成，每个节点包含对象。通过将文档表示为DOM，然后操作DOM，可以以编程方式更改网页（例如，结构、样式或内容）[5]。

<img width="438" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/b88c232a-1c66-406b-8305-5e46531e3601">

#### 2.1.2 XML路径语言

XML路径语言（XPath）以路径符号提供了一种灵活的方法来寻址XML或HTML对象的部分。XML路径语言（XPath）表达式可用于在HTML文件的DOM树中导航，而无需依赖DOM核心特性，例如Document和Node接口，这些接口提供了getElementById()和ChildNodes等方法和属性 [9]。图2.2显示了应用于同一HTML对象的两个XPath表达式的示例。

<img width="498" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/83f5f693-eadc-45e1-947e-620aeedae10e">

#### 2.1.3 JavaScript对象表示法

JavaScript对象表示法（JSON）是一种轻量级的与语言无关的数据格式 [10]。JSON具有易于阅读和编写的文本格式，如图2.3所示。它基于两种结构：一组键/值对和一个有序列表。键/值对的集合称为对象，其中键/值对在左括号和右括号之间列出，键/值之间用冒号分隔。有序列表可以包含多个对象。

```json
[
  {
    ”name”: ”Alice”,
    ”age” : 25
  },
  {
    ”name”: ”Bob”,
    ”age” : 26,
    ”height”: 174.5
  }
]
```

### 2.2 深度学习

深度学习是机器学习的一个子集，其中建模并训练神经网络，试图模拟人脑在学习过程中的行为 [11]。在深度学习中，需要较少的数据预处理，可以使用非结构化数据，如文本和图像。使用深度学习的一个显著优势是自动特征提取，机器决定哪些特征是相关的，而无需依赖人类专家。使深度学习网络“深”的主要因素包括层中神经元的数量、这些层之间连接的复杂方式以及训练网络所需的大量计算能力 [12]。
以下小节介绍了几个深度学习概念，这些概念对理解模型架构很重要，具体包括卷积神经网络（CNNs）（第2.2.1节）、循环神经网络（RNNs）（第2.2.2节）和变压器（第2.2.3节），以及迁移学习的概念（第2.2.4节）。

#### 2.2.1 卷积神经网络

卷积神经网络（CNNs）是深度网络的主要架构之一，其目标是通过利用卷积进行特征检测，学习数据中的高阶特征。这通过对两组信息应用数学运算来实现 [12]。CNNs主要用于机器视觉（例如，图像分类），但也适用于文本分析。在建模数据（如图像）时，CNNs具有较高的计算效率，否则在全连接网络中可能导致连接数量激增。
主要的三个层组包括：输入层、特征提取层和分类层，如图2.4所示。架构各层之间的主要区别在于特征提取层，它包含两种类型的层：卷积层和池化层。

<img width="455" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/ff250a43-04cc-480e-9c33-fb3817ac2fa3">

卷积层在数据中寻找特征，通过对输入应用滤波器将这些特征组合成高阶特征。图2.5中显示了一个这样的滤波器，其核（即，滤波器）向量的权重为[1/3, 1/3, 1/3]。在一层中可以应用多个不同的滤波器。在应用滤波器后，激活函数用于决定哪些神经元应该被激活并传播其值。两种这样的激活函数是修正线性单元（ReLU）和高斯误差线性单元（GELU）。线性函数ReLU对于所有非负输入都输出相应的输入，否则输出零，即max(0, x)。而GELU [13] 是一个更复杂的非线性函数，可以看作是ReLU的一个更平滑的版本。

<img width="429" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/ec51720d-1956-41cb-80f1-3a7827281ed2">

池化层在卷积层之后使用，以减小（即，下采样）数据表示的空间大小。这有助于减少网络记忆训练数据的自由度（即，过拟合），而是迫使其进行学习泛化；因此，在未见过的数据上表现更好。最大池化是其中的一种常见变体，它选择滤波器区域中的最大值。

#### 2.2.2 循环神经网络

RNN与其他类型的神经网络有所不同，因为它们具有对数据的时间维度（即，时间依赖性）进行建模的能力。RNN在每个输入（即，时间步）之间保留状态，它使用这些状态对数据进行建模，然后将状态传递到下一个时间步。这种时间反馈使模型能够捕捉上下文，特别是对于需要基于当前和先前输入生成/推断序列的敏感数据，如语言、音频和文本 [12]。
长短时记忆（LSTM）[14]是最常见的RNN架构之一。其主要优势在于它能够在时间步之间保持内存不变。这种特性使其能够克服梯度消失问题，即模型由于模型（即，权重）的更新变得非常小，导致模型停止学习，无法进一步捕获任何输入。

#### 2.2.3 变压器

变压器是由Ashish Vaswani等人在他们的论文《Attention is all you need》[15]中提出的最新架构之一。它是一种完全基于注意机制而非循环或卷积的架构。与具有顺序性质的循环相比，这种结构在训练期间具有更大的并行性，其中新的隐藏状态是作为过去状态的函数而生成的。
变压器架构基于一个编码器-解码器结构，包括编码器和解码器堆栈，每个堆栈由六个相同的层组成。编码器堆栈负责将输入的符号序列映射为连续表示。解码器堆栈生成一个符号序列，其中每次生成一个符号，并在下一生成步骤中用作额外的输入。

#### 2.2.4 迁移学习

迁移学习是通过从相关领域传递信息来改进某一领域中的学习者的一种方式。在神经网络中，由于需要更大的数据集来训练网络以避免过拟合 [16]，迁移学习可以发挥重要作用，特别是在训练集有限的情况下。与从头开始训练一个模型不同，可以利用已经使用与目标域相关的更大数据集进行训练的模型，用于任务如文本情感分析和图像分类 [17]。迁移学习可以在包含两个阶段的学习框架中形式化：预训练和微调 [18]。
预训练阶段包括捕捉一个或多个任务的知识。这可以通过大规模未标记的语料库来学习良好的表示，然后在其他任务中使用这个表示。预训练的一些优势包括学习通用语言表示、更好的模型初始化以及在小数据集上防止过拟合的正则化效果 [16]。微调阶段使用预训练模型，并进一步使用代表特定问题的较小数据集进行所谓的下游（即，目标）任务的训练。

### 2.3 评估指标

该模型的三个评估指标将是：精确度、召回率和F分数。在关注分类性能的机器学习应用中，这些是关键指标。
精确度衡量正类别的预测值，同时避免将负类别错误地分类为正类别 [19]。具体而言，正确定义的预测中实际正确的比例：

$$
\[ \text{精确度} = \frac{\text{真正例}}{\text{真正例 + 假正例}} \quad (2.1) \]
$$

召回率衡量正类别的预测值，同时避免将正类别错误地分类为负类别 [19]。具体而言，正确定义的预测与所有实际正类别的比例：

$$
\[ \text{召回率} = \frac{\text{真正例}}{\text{真正例 + 假负例}} \quad (2.2) \]
$$

F分数，即F1分数，是精确度和召回率之间的调和平均值。基于F-beta分数，其中精确度和召回率根据beta值具有不同的权重 [19]。当beta为1时，精确度和召回率具有相等的权重（即，相等的重要性）。

$$
\[ \text{F-beta} = \frac{(\beta^2 + 1) \times \text{精确度} \times \text{召回率}}{\beta^2 \times \text{精确度} + \text{召回率}} \quad (2.3) \]
$$
$$
\[ \text{F1} = 2 \times \frac{\text{精确度} \times \text{召回率}}{\text{精确度 + 召回率}} \quad (2.4) \]
$$

在优先考虑精确度或召回率的情况下，高度依赖于具体情境。由于它们通常对彼此产生相反的影响 [20]，最大化其中一个可能会降低另一个。在医学诊断中，假负例可能比假正例更昂贵（即，致命），因此在这种情况下可能更重要，应相应地加以权重。

### 2.4 相关工作
存在一些可以构建在其基础上的相关工作。具体而言，有关网络数据提取文献的调查（第2.4.1节），语言模型Bidirectional Encoder Representations from Transformers（BERT）（第2.4.3节）以及SOTA模型MarkupLM（第2.4.5节），本论文使用它们作为基础。

#### 2.4.1 网络数据提取调查
Emilio Ferrara等人进行了一项调查，全面概述了网络数据提取领域的文献，并为网络数据提取应用提供了分类框架 [21]。他们确定了两种主要的算法方法：树匹配和机器学习算法。

##### 2.4.1.1 树匹配算法
树匹配算法利用Web页面的半结构化特性，以HTML的形式表示为带有标签的有序根树，即DOM树。

这些类型的算法使用XPath语法处理DOM树中的特定元素。它们依赖于XPath表达式，以找到两个文档之间相似树的所谓树编辑距离匹配。类似于字符串编辑距离问题，两个有序树可以通过尽可能少的操作（即，节点删除、插入或替换）来相互转换以匹配。简单的树匹配算法 [22]是树编辑距离匹配问题的高效且易于实现的解决方案 [23]。

##### 2.4.1.2 机器学习算法
机器学习算法是一种适用于具有不同结构的多个网站的领域特定提取的良好方法。这些算法依赖于手动标记的网站，以获取领域专业知识，一些最早使用机器学习的系统包括WIEN [24]、Rapier [25]和WHISK [26]。

WIEN专注于归纳学习技术，以自动生成包装器。生成的规则可能类似于“忽略所有字符，直到找到第一个'.'并提取餐厅名称，该字符串以第一个':'结束。然后，再次忽略所有字符，直到找到'('并提取以')'结束的字符串。” [27]。类似这样的规则会在存在多个对象的情况下重复，直到无法与其他对象匹配。

Rapier使用有限的句法和语义信息学习规则，而无需在文档之前进行解析或后处理。规则分为三种模式：前填充器、填充器和后填充器。其中前填充器和后填充器充当左右分隔符，而填充器模式描述目标信息结构。

WHISK生成可以处理各种结构的文档（从自由文本到HTML）的规则。这些规则是一种特殊类型的正则表达式（即，尝试与输入文本匹配的模式），由两个组件组成。第一个组件负责确定短语必须处于其中以使其相关的正确上下文，而另一个指定要提取的短语的哪些部分。

#### 2.4.2 网页表提取调查
Shuo Zhang等人进行了一项调查 [28]，研究了有关网页表提取的文献。其目的是确定和描述几个网页表提取任务及其相互依赖关系。他们确定了六个主要类别，用于对文献进行分类。这些类别包括：表提取、表解释、表搜索、问题回答、知识库增强和表增强。

他们定义了一个表由以下元素组成：页面标题、标题、列、单元格、行、列和实体。提出了一种表分类方案，通过两个维度内容和布局来区分表。

##### 2.4.2.1 表提取
表提取是在网页上检测和提取表格，然后以一致的格式存储的过程。在网上提取表格的第一步是过滤掉“不好的”表格（例如，用于布局或格式目的的表格）。这通常通过关系表分类来完成，以识别包含关系数据的表格。在这里，可以使用具有布局或内容类型特征的机器学习分类器。布局特征可以是行数、列数或平均单元格字符串长度。而内容类型特征可以是表体中非字符串数据的百分比、带

有数字字符的单元格的比例，或包含 `<span>` 标签的单元格的比例。类似的方法也可以用于表头检测和表类型分类，前者检测表是否包含标题行或列，而后者根据预定义的分类法对表进行分类。

##### 2.4.2.2 表解释
表解释旨在发现网页上表格的语义，以便对表格中的数据进行智能处理。使用分类法来了解表列的含义以及它们是否与其他列相关。主要的任务有列类型识别、实体链接和关系提取。

列类型识别涉及确定列类型并定位核心列（即，主体列），通常是最左边的列。实体链接是指检测实体（例如，人物、组织和地点），这对于揭示语义至关重要。关系提取旨在将一对列与其内容之间的关系关联起来。

##### 2.4.2.3 表搜索
表搜索通过关键字查询返回带有排名列表的表，其中查询可以是一个表或多个关键字。主要有基于关键字和基于表的两种搜索类型。基于关键字的搜索返回给定关键字查询的表的排名列表。

##### 2.4.2.4 问题回答
问题回答试图使用表格中的结构化数据回答自然语言处理问题。使用表格回答问题的主要挑战是将非结构化查询与表格中的结构化信息匹配。将查询解析为形式化表示的任务称为语义解析，其中生成逻辑表达式，可在知识库上执行。

##### 2.4.2.5 知识库增强
知识库增强使用表格数据来探索、扩展或构建知识库。知识探索可以在具有属性搜索查询或实体关系查询的表格上进行。通过使用知识库进行注释，然后从表格中提取信息，可以扩展现有的知识库。如果表格包含丰富的信息，它可以转化为新的知识库。

##### 2.4.2.6 表增强
表增强通过添加附加数据扩展现有表格。它可以分为三个任务：行扩展、列扩展和数据完成。行扩展通常应用于水平关系表。相反，列扩展通常通过查找相似的表格，然后评估这些表格中的列标题和值来添加额外的列。数据完成可以应用于整个列，通过匹配来自其他表格的类似列，或在单个单元格上使用机器学习算法，例如k最近邻或线性回归。

##### 2.4.3 双向编码器表示转换器
Jacob Devlin等人提出了一种名为BERT的新语言表示模型 [29]。预训练语言模型已被证明可以在句子和标记级别的任务上提高几种自然语言处理问题。然而，以前的技术限制了预训练模型的体系结构选择，使其能够联合条件化左侧和右侧（即双向）上下文，这对于标记级别的任务如问答至关重要。BERT通过利用Transformer体系结构（第2.2.3节）和两个预训练目标实现了双向预训练。

该体系结构是一个多层双向Transformer编码器，并具有需要最小更改用于最终下游体系结构的属性。输入表示可以处理单个和多个句子作为输入序列，并以三种方式嵌入：令牌、段和位置嵌入。这三种嵌入求和以表示输入嵌入，如图2.6所示。一个令牌可以是三种情况之一：特殊的序列开始令牌（[CLS]），一个单词或一个分隔令牌（[SEP]）以区分句子。特定于序列中的令牌所属的段嵌入（例如，句子A或B）。位置嵌入编码了序列中令牌的位置。

<img width="499" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/a80b8b0a-5db7-48fc-a524-5d582cf98928">

两个目标，遮蔽语言建模（MLM）和下一句预测（NSP），在预训练期间被使用。MLM通过随机遮蔽输入标记的一部分，然后训练模型预测被遮蔽标记，使模型学习双向表示。NSP使模型学习两个句子之间的关系。选择两个句子A和B，其中句子B一半的时间被随机替换，要求模型预测句子B是否跟随句子A。

BERT使用两个数据集进行预训练：BooksCorpus [30]（800M字）和English Wikipedia（2500M字）。BERT是第一个基于微调的表示模型，在多个标记和句子级任务上取得了SOTA结果，如通用语言理解评估（GLUE）[31]、斯坦福问答数据集（SQuAD）[32]和带有对抗生成的情境（SWAG）[33]。

#### 2.4.4 SimpDOM
Yichao Zhou等人探索了在相同垂直领域内从多个网站提取数据的可能性 [34]。他们提出的模型∗，称为SimpDOM，在使用Few-Shot Learning（FSL）准确提取未见网站的数据时取得了SOTA结果。

SimpDOM模型的主要思想是专注于HTML页面的DOM树表示，并为每个变量节点构建丰富的表示。该方法避免了昂贵的网页呈现过程，利用DOM树中节点属性值的语义。

该架构由DOM树简化模块、离散特征模块和文本编码器组成。DOM树简化模块提取具有不同值的所有节点的上下文（因为在数据点之间具有相同值的节点不感兴趣）。上下文是其友好节点（即附近节点）的特征。离散特征模块通过添加额外的离散特征（例如XPath、叶节点类型和相对节点位置）来增强节点表示。文本编码器是CNN-LSTM的组合，对字符和单词级特征进行编码。

使用Structured Web Data Extraction（SWDE）数据集对SimpDOM进行评估。该数据集最初由郝强等人创建 [35]，包含来自80个不同网站的124,000个标记页面，分为八个垂直领域（例如汽车、图书和电影），每个领域包含3到5个感兴趣的属性（例如标题和作者）。在每个垂直领域中，使用10个网站中的5个作为种子站点（即训练集中的站点），SimpDOM实现了93.75的平均F1分数。

SimpDOM的作者选择使用一个基于Global Vectors for Word Representation（GloVe）[36]架构训练的，包含60亿标记的著名预训练词嵌入来初始化他们的模型。

#### 2.4.5 MarkupLM

Junlong Li等人研究了创建一个模型∗，能够解决多个文档理解任务，适用于视觉丰富的标记文档，如HTML和XML文件 [37]。任务包括文档理解、类型分类和视觉问答。通过利用DOM树，可以对文档的不同元素之间建模位置关系，而不是使用显式的2D表示，这对文档渲染的设备高度依赖。通过使用DOM树建模位置关系而不使用渲染的2D可视化，简化了预训练，同时仍然利用了文档布局。

BERT [29]体系结构被用作编码器，其中嵌入层扩展了额外的输入XPath嵌入。然后，该模型通过三个主要目标进行预训练：遮蔽标记语言建模（MMLM），节点关系预测（NRP）和标题页匹配（TPM）。MMLM是MLM的扩展，通过使用文本和标记作为输入，遮

<img width="496" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/03eed6a4-bfc1-40bc-a231-8ed78c21932d">

#### 2.4.6 DOM-LM

邓翔等人提出了一种能够解决类似文档理解任务的模型，与MarkupLM一样，利用了DOM树表示法，就像以前的工作所做的一样 [39]。该模型基于BERT（与MarkupLM相同），其参数是从预训练的BERT模型（对非结构化文本进行预训练）中初始化的，然后进一步训练以捕获HTML文档的结构和布局信息。该模型以两个目标进行预训练：遮蔽节点预测（MNP）和遮蔽标记预测（MTP）。MTP类似于BERT中执行的MLM目标（以及MarkupLM中的修改变体MMLM）。MNP通过不仅遮蔽输入标记而且遮蔽整个节点来进一步概括模型，以迫使模型学习树级上下文化，并对布局具有整体视图。

该模型的主要方法是将文档编码为一组子树，其中嵌入了位置信息，并采用了自监督预训练。首先通过去除与网页结构和语义无关的所有DOM节点（例如，<script> 和 <style> 元素），然后将树分割成子树来构建一组子树。分割是通过在整个DOM树上应用具有固定步长（即单位）的滑动窗口来完成的。滑动是这样进行的，以便在同一直接周围的节点被捕获在同一子树中。

在培训过程中使用的数据量与SimpDOM和MarkupLM中使用的数量有所不同。DOM-LM仅使用了10％的种子站点（2和5）的数据，而不是所有的数据，这与SimpDOM和MarkupLM不同。结果可见于表2.2。

<img width="494" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/6c4a8b89-49a5-4e4b-ae9d-533391a8c3b4">

### 2.5 总结

本章在第2.1节介绍了相关的关键网络技术DOM和XPath，第2.2节介绍了深度学习架构，如RNN和Transformer，第2.3节介绍了度量标准，第2.4节介绍了相关工作，如BERT和MarkupLM。

## 3、方法

在本章中，介绍了研究方法。研究过程在第3.1节中描述，数据收集过程在第3.2节中描述，最后，在第3.3节中描述了实验设计和评估框架。

### 3.1 研究过程

该论文遵循设计科学研究过程，该过程可以分为六个活动，如Ken Peffers等人在[40]中提出的。这些活动包括：问题识别和动机、解决方案目标、设计和开发、演示、评估和沟通。根据这个过程，研究人员并不被期望按顺序进行这些活动，而是取决于所选择的方法（例如，问题中心或目标中心）。
问题识别和动机的活动涉及具体说明研究问题并证明解决方案的价值。这可以通过获取有关问题状态和解决方案重要性的知识来实现。
解决方案目标指的是解决方案的定量或定性目标。这些目标通常涉及所期望的解决方案，该解决方案应更好或解决未解决的问题。在这个阶段，需要了解当前解决方案。

设计和开发涉及创建一种工件解决方案（例如，构造、模型、方法）。在创建解决方案之前，需要理解理论，决定工件的功能和架构。

演示工件在解决问题时的有效性。这可以通过实验、模拟、证明或案例研究来完成。

评估是衡量工件支持问题解决的效果。通过使用相关的度量和分析技术，可以确定工件的有效性，并作为迭代回活动3（设计和开发）的基础，以尝试在可行的情况下改进工件。

沟通是最后的活动，在其中整个过程都被记录在研究论文中。这包括问题及其重要性、工件及其效用、研究的设计以及与社区的相关性。

问题是在主机公司识别的，解决方案得到了证明。随后进行文献研究以更好地了解问题的状态和当前的解决方案。然后确定了建立在当前解决方案基础上以在新环境中解决问题的目标。工件的评估类似于相关工作中的模型。最后，在这篇论文中记录了整个过程。

### 3.2 数据收集
汽车保险数据来自几家瑞典保险公司，其中主机公司目前使用和维护手工制作的包装器。数据将采用HTML的形式，并附有JSON文件，表示由包装器提取的值的基本事实。敏感用户信息在用于模型之前被混淆。仅使用具有不为空的JSON对应文件的HTML文件。如果JSON中没有提取的值，则假定包装器失败或用户在给定网站上没有保险。

### 3.3 实验设计

实验设计遵循相关工作中使用的设计[34, 35, 37]，在k个种子站点上对模型进行微调，然后在其余的n−k个站点上进行评估（即零-shot学习）。评估指标是页面级F1分数，最终F1分数是每个k的所有排列的平均值。模型使用一个瑞典和一个英文预训练模型初始化，然后进行实验性评估和比较。

模型在深度学习的Amazon Machine Image（AMI）[41]上进行训练，实例类型为G4dn [42]，具有以下规格：1个Nvidia Tesla T4图形处理单元（GPU），8个Intel Cascade Lake虚拟中央处理单元（CPU），32GB内存。

## 4、实现

本章描述了获取数据集和修改针对性网站的开源模型的步骤。数据预处理步骤在第4.1节中介绍，MarkupLM模型的实施和修改细节在第4.2节中给出。

### 4.1 数据预处理
在使用模型训练数据之前，数据需要进行处理。从公司收到的数据包括三种类型的文件：HTML、JSON和文本（日志）文件。HTML文件包含用户的保险数据，并且仅限于包含单一保险的页面（省略包含多个保险的HTML文件）。JSON文件包含目标属性的提取数据，公司手动开发的包装器执行提取。最初的计划是使用JSON数据作为数据集的基本事实，但由于某些值是使用正则表达式进行转换的，因此与HTML中的文本不是精确匹配，这是不可能的。日志文件包含包装器整个执行流程的日志消息。幸运的是，提取的值在转换之前被转储到日志中，因此可以使用日志中的数据作为基本事实。
模型要提取的属性数量被限制为三家选定公司的较大部分数据中出现的五个属性：Trygg-Hansa、If和Moderna。数据集中仅使用包含所有属性的数据点。这五个属性是：保险覆盖类型、保险单号、年度保费金额、续保日期和车辆注册号。每家公司的数据点数量如表4.1所示，每个属性的统计信息如表4.2所示。

<img width="427" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/d24d2b15-9220-40ac-ad36-81a77e4b61a5">

<img width="527" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/d4d515e7-8d6e-4d9b-81b2-474df1cea7ba">

### 4.1.1 基本事实
通过解析包含以JSON格式提取和未经处理的值的日志，生成了包含HTML和基本事实文件的数据集。基本事实JSON中的对象通过保险类型（即汽车保险）进行过滤。如果网页上有其他类型的保险或超过一种汽车保险，则会省略数据点。之所以这样做，是因为三家公司中有两家公司在单个网页上显示客户的所有保险，而在DOM树中没有区分。这意味着所有保险都以相同的HTML列表对象的方式列出，而且在其中没有任何特定顺序，这样模型就很难学习上下文如何区分一个保险对象和另一个保险对象。

### 4.1.2 数据模糊化
在数据集可以传输到AMI并在模型中使用之前，必须模糊化可以与特定个人关联的所有敏感数据。属性保单持有人、地址、保险单号和车辆注册号都被替换为随机生成的值。使用网站www. fejk.se生成虚假姓名、个人身份号码和地址，而使用Python脚本生成保险和车辆注册号。

## 4.2 实施
所使用的模型基于开源模型MarkupLM [37]，该模型在第2.4.5节中有描述。此模型使用Python编写，使用Pytorch机器学习框架 [43] 和提供API以轻松下载和训练SOTA预训练模型的Transformers库 [44]。模型的执行步骤如图4.1所示。
第一步将HTML文件打包成适当的Python数据对象，然后将它们序列化为单个文件，这是使用pickle库 [45] 完成的。第二步创建了每个HTML文件与相应基本事实之间的映射，并将其作为pickle文件分别存储在每个网站上。最后一步使用所有种子站点的排列训练和评估模型，其中在训练之前使用预训练的模型初始化。模型在多个周期（即在训练集中循环）中进行训练，最后在未被种子化的每个网站上进行评估。

<img width="261" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/a30c1520-1efe-46c8-a94b-aefa952e42a1">

大部分的修改发生在数据准备和评估步骤。数据准备步骤，即HTML与基本事实之间的映射，需要修改以处理瑞典字符，并处理网站的边缘情况，其中基本事实的值并未单独位于正确的节点中，而必须使用正则表达式进行匹配。评估步骤通过更详细的记录和在每个周期后进行模型评估，引入了早停机制，以在训练损失不再降低时终止训练。
在对保险数据集进行微调之前，MarkupLM模型是使用预训练模型初始化的。论文中使用的两个预训练模型是MarkupLMLARGE和来自瑞典国家图书馆的瑞典BERT模型。

### 4.2.1 MarkupLM-LARGE
MarkupLM论文的作者们 [37] 也开源了两个预训练模型，MarkupLMBASE和MarkupLMLARGE。他们首先在Common Crawl（CC）数据集∗的 2400 万个英语网页上对MarkupLM模型进行了预训练，该数据集使用了原始论文作者发布的Robustly Optimized BERT pretraining Approach (RoBERTa)模型进行初始化 [46]。然后在SWDE数据集上对该模型进行了微调。MarkupLMLARGE在SWDE上的性能见表2.1。

### 4.2.2 瑞典BERT
瑞典国家图书馆（瑞典文：Kungliga biblioteket）于2020年发布了三个基于BERT和A Lite BERT (ALBERT) [47] 的预训练瑞典语言模型。这些模型是在一个18,341 MB的瑞典文语料库上进行训练的，该语料库由报纸、政府报告、法定电子存档、社交媒体评论和维基百科的文本组成。他们的预训练BERT模型名为KB-BERT，用于在微调之前初始化模型。初始化是通过使用Transformers库实现的，加载托管在AI社区站点Hugging Face [48] 上的预训练模型，该站点还负责Transformers库。表4.3显示了两个预训练模型的一些关键模型配置参数，这些参数遵循原始BERT论文的设置（除了词汇表大小）。

<img width="434" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/05e280a7-f189-49d5-82ac-9bb5631dedfb">

## 5、结果

这一章介绍了模型评估的结果。以精确度、召回率和F1分数为指标，展示了两个模型的页面级结果，并详细分析了属性级结果。

两个模型的总体结果显示在表5.1中。使用KB-BERT初始化的模型在训练过程中使用一个种子站点时，最佳F1分数为41.4，当使用两个种子站点时为47.3。使用MarkupLMLARGE初始化的模型在使用一个种子站点进行训练时，最佳F1分数为80.2，使用两个种子站点时为88.9。在评估过程中变化的最佳设置分别在表5.2和表5.3中显示，对应着一个和两个种子站点。

<img width="517" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/01b15133-6bd0-4cdc-ab57-c92db267599e">

<img width="499" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/da7cffd5-148d-4ef0-8d24-6d1e4258e5ea">

使用KB-BERT初始化的模型每个属性的F1分数显示在表5.4中。该模型无法学习保单号的表示，同时在处理年度保费属性时也存在一些问题。

<img width="496" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/029d6b49-02a7-471d-8d09-a28180880c8d">

使用MarkupLMLARGE初始化的模型每个属性的F1分数显示在表5.5中。该模型对大多数属性学到了一个表示，但在处理保险类型属性时并不那么成功。

<img width="497" alt="image" src="https://github.com/holmofy/blog.hufeifei.cn/assets/19494806/13558459-3124-4480-998b-3cb114459e0d">

## 6、讨论

这一章将讨论第5章中提出的结果，同时也将包括对实现目标（第1.4节）和研究问题（第1.2节）的探讨。

### 6.1 结果

结果表明，尽管MarkupLMLARGE模型并不“理解”瑞典语，而是学会了HTML的一般结构，但它在预训练模型的强大性上表现出色。这一事实事后看来并不令人惊讶，因为选择的五个属性中有四个不特定于瑞典语。

使用MarkupLMLARGE初始化的模型在三个属性上表现非常好：保险单号、续保日期和车辆注册号，在使用一个和两个种子站点时，F1分数均超过94%。保险单号具有特定格式（例如123 456 789、AB00123456.2.3和123456-78），每家公司有时甚至有几种格式。续保日期始终以YYYY-MM-DD的格式呈现，而车辆注册号有两种格式之一，即ABC123或ABC12D。年度保费这个属性的得分相对较低，这很可能是因为它在不同公司之间的格式不同，有时包含瑞典语词汇（例如“年”这个瑞典词）。然而，即使对于只包含瑞典语词汇的保险类型属性，该模型在某种程度上也成功地学到了，特别是在将种子站点数量从一个增加到两个时。

与MarkupLMLARGE初始化的模型相比，使用KB-BERT初始化的模型表现相对较差。使用一个和两个种子站点时，得分最高的属性是覆盖类型和续保日期，F1分数均超过71%。覆盖类型通常是“halvförsäkring”、“helförsäkring”、“trafikförsäkring”或其变体中的一个词，这是模型在这方面优于MarkupLMLARGE初始化的模型的唯一属性。令人惊讶的是，该模型无法学习保险单号的表示，对于使用一个和两个种子站点时的F1分数均低于4%。这可能是由于不同公司之间的格式不一致，再加上一些格式包含标点符号，这可能使学习变得更加困难，因为BERT将标点符号视为输入序列中“句子”的分隔符。

另一个方面是，瑞典BERT使用了较小的BERTBASE的参数，而MarkupLMLARGE使用了较大的BERTLARGE，它们在隐藏层的数量、隐藏层的大小以及每个隐藏层的注意头的数量上存在差异（如表4.3所示）。然而，在原始的BERT论文[29]中，BERTLARGE在GLUE基准上的得分约高出BERTBASE约3%，而在使用5个种子站点时，MarkupLMLARGE在SWDE数据集上的得分比MarkupLMBASE高出约1.5%。这应该表明，性能的这种大差异不能仅通过网络大小的差异来解释。

在表5.2和表5.3中显示的设置是为了寻找最佳精度而进行的调整的设置。在表现最佳的运行之间，最常见的批量大小为2。这似乎是合理的，因为数据集很小，数据集中最大的公司有100个数据点，而最小的公司只有23个。用于上下文的前面节点的最佳数量是4。训练时间的巨大差异可以通过两个预训练模型之间的参数差异（主要是层数和层大小）来解释。作为参考，BERT论文的作者使用4个云张量处理单元（TPU）[49]对BERTBASE进行了训练，分别对BERTLARGE进行了16个云TPU的训练，分别进行了4天的训练。

### 6.2 指标选择

在网络数据提取领域存在多种度量标准，然而，本论文专注于三个准确性指标：召回率、精确度和F分数。尽管这些指标并非完美[50]，但它们在机器学习评估中被广泛使用。由于大多数相关工作都在使用这些指标，尽管它们在处理负面例子时表现不佳，但它们被选择了。F分数的beta值被选择为1（即F1分数），因为没有（通过主机公司）明确要求将精确度优先于召回率，反之亦然。

### 6.3 达到目标

在第1.4节中提到的四个子目标是：数据集获取、模型识别、模型修改和模型评估。根据结果，可以说这四个子目标都已经实现。

收集到的数据比预期的要小，无论是从公司和网站的数量上还是从每家公司和网站的数据点数量上。然而，尽管数据集相对较小，MarkupLMLARGE模型取得了相对较好的分数，展示了预训练模型的强大性能。

模型的识别是成功的，并导致了SOTA模型MarkupLM的产生，尽管最初选择的是SimpDOM（第2.4.4节），这在第7章中作为一个局限性进一步讨论。模型修改按照最初的计划使用了一个预训练的瑞典BERT

模型，尽管其性能不及MarkupLM作者发布的预训练模型。最后，该模型使用了两个不同的预训练模型，在使用一个和两个种子站点时进行了评估。

### 6.4 回答研究问题

最初提出的研究问题是：

一个带有监督学习的网络数据提取模型在从未见过的瑞典保险网站中提取信息的能力如何？

最初的假设是这样的系统必须对瑞典语有一个良好的表示才能有效，因此需要使用在瑞典语上进行预训练的语言模型进行探索。然而，结果表明，这种对语言的依赖性表示不一定是必要的，至少当大多数属性不是语言特定的或者在其上下文中不被语言特定的文本包围时。此外，即使对于语言特定的属性，例如保险类型，也是在既预训练于英语语言又预训练于网页的模型表现得更好，而后者仅在瑞典语上进行了预训练。

对研究问题的回答是这样的模型实际上在瑞典保险网站上表现良好，达到了与在SWDE数据集上的SOTA模型相似的结果（如表2.1所示）。

## 7、结论与未来工作

### 7.1 结论
最初的假设认为在瑞典网站上，以英语数据为基础的模型性能较差。然而，结果表明，如果属性不包含特殊的瑞典字符，这样的模型表现良好。这种模型的学习强调HTML文档结构，特别是当值嵌入在结构化格式（如表格）中时。虽然没有达到100%的准确性，但在手动创建的包装器因站点故障或HTML更改而失败的情景下，该模型可能会很有价值。

### 7.2 限制

显著的限制包括有限的可用数据量，导致较少的候选公司可供论文使用。数据的倾斜和耗时的预处理影响了模型的探索。尝试复制 SimpDOM 的努力没有成功，妨碍了与 MarkupLM 的潜在比较。

### 7.3 未来工作

**尚未完成的工作**
  * 探索提取不可靠出现在大多数客户数据中的属性。
  * 调查在网页上显示多个保险政策的情况。
  * 为潜在的准确性提升调整学习率和丢失概率等超参数。

**下一个明显需要做的事**
  * 评估在其他行业培训模型的好处。
  * 探索少样本学习，对目标网站进行更多的手动标记。
  * 考虑混合方法，利用开源模型提取特定属性。

### 7.4 反思
从经济角度来看，所探讨的模型减少了在相同行业内设置数据提取的手动工作，并减少了对覆盖的网站进行的维护。与模型培训相关的计算成本可以通过使用开源预训练模型来缓解。伦理考虑涉及根据政府政策和法规处理客户数据。敏感数据的混淆是一种方法，但自动确定哪些数据是敏感并需要混淆是本工作范围之外的问题。

引用：

[1] R. Kosala and H. Blockeel, “Web mining research: a survey,” ACM SIGKDD Explorations Newsletter, vol. 2, no. 1, pp. 1–15, Jun. 2000. doi: 10.1145/360402.360406. [Online]. Available: https://dl.acm.org/doi/10.1145/360402.360406 [Page 1.]
[2] J. Wang and F. H. Lochovsky, “Data extraction and label assignment for web databases,” in Proceedings of the twelfth international conference on World Wide Web - WWW ’03. Budapest, Hungary: ACM Press, 2003. doi: 10.1145/775152.775179. ISBN 978-1-58113-680-7 p. 187. [Online]. Available: http://portal.acm.org/citation.cfm?doid=775152.775179 [Page 1.]
[3] A. Troestler and H. P. Lee, “The adaptation and standardization on websites of international companies : Analysis and comparison from websites of United States, Germany and Taiwan,” Ph.D. dissertation, Linköping University, 2007. [Online]. Available: http://urn.kb.se/resolve?urn=urn:nbn:se:liu:diva-9801 [Page 1.]
[4] S. Flesca, G. Manco, E. Masciari, E. Rende, and A. Tagarelli, “Web wrapper induction: a brief survey,” AI Communications, vol. 17, no. 2, pp. 57–61, Apr. 2004. [Online]. Available: https://dl.acm.org/doi/10.5555/1218702.1218707 [Page 2.]
[5] “Document Object Model (DOM),” Dec. 2021. [Online]. Available: https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model [Pages 2 and 5.]
[6] K. Lerman, S. N. Minton, and C. A. Knoblock, “Wrapper Maintenance:A Machine Learning Approach,” Journal of Artificial Intelligence Research, vol. 18, pp. 149–181, Feb. 2003. doi: 10.1613/jair.1145. [Online]. Available: https://jair.org/index.php/jair/article/view/10325 [Page 2.]
[7] Hevner, March, Park, and Ram, “Design Science in Information Systems Research,” MIS Quarterly, vol. 28, no. 1, p. 75, 2004. doi: 10.2307/25148625. [Online]. Available: https://www.jstor.org/stable/10.2307/25148625 [Page 3.]
[8] R. Baumgartner, W. Gatterbauer, and G. Gottlob, “Web Data Extraction System,” in Encyclopedia of Database Systems, L. Liu and M. T.Özsu, Eds. Boston, MA: Springer US, 2009, pp. 3465–3471. ISBN978-0-387-39940-9. [Online]. Available: http://link.springer.com/10.1007/978-0-387-39940-9_1154 [Page 5.]
[9] “XPath,” Jan. 2022. [Online]. Available: https://developer.mozilla.org/en-US/docs/Web/XPath [Page 6.]
[10] “Introducing JSON.” [Online]. Available: https://json.org/json-en.html [Page 6.]
[11] “What is Deep Learning?” May 2020. [Online]. Available: https://www.ibm.com/cloud/learn/deep-learning [Page 7.]
[12] J. Patterson and A. Gibson, Deep learning: a practitioner’s approach, 1st ed. Sebastopol, CA: O’Reilly, 2017. ISBN 978-1-4919-1425-0 [Pages 7, 8, and 9.]
[13] D. Hendrycks and K. Gimpel, “Gaussian Error Linear Units (GELUs),” arXiv:1606.08415 [cs], Jul. 2020. [Online]. Available: http://arxiv.org/abs/1606.08415 [Page 9.]
[14] S. Hochreiter and J. Schmidhuber, “Long Short-Term Memory,” Neural Computation, vol. 9, no. 8, pp. 1735–1780, Nov. 1997. doi:10.1162/neco.1997.9.8.1735. [Online]. Available: https://direct.mit.edu/neco/article/9/8/1735-1780/6109 [Page 9.]
[15] A. Vaswani, N. Shazeer, N. Parmar, J. Uszkoreit, L. Jones, A. N. Gomez, ff. Kaiser, and I. Polosukhin, “Attention is All you Need,” in Advances in Neural Information Processing Systems, I. Guyon, U. V.Luxburg, S. Bengio, H. Wallach, R. Fergus, S. Vishwanathan, and R. Garnett, Eds., vol. 30. Curran Associates, Inc., 2017. [Online]. Available: https://proceedings.neurips.cc/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf [Page 10.]
[16] X. Qiu, T. Sun, Y. Xu, Y. Shao, N. Dai, and X. Huang, “Pretrained models for natural language processing: A survey,” ScienceChina Technological Sciences, vol. 63, no. 10, pp. 1872–1897, Oct. 2020. doi: 10.1007/s11431-020-1647-3. [Online]. Available: https://link.springer.com/10.1007/s11431-020-1647-3 [Page 10.]
[17] K. Weiss, T. M. Khoshgoftaar, and D. Wang, “A survey of transfer learning,” Journal of Big Data, vol. 3, no. 1, p. 9, Dec. 2016. doi: 10.1186/s40537-016-0043-6. [Online]. Available: http://journalofbigdata.springeropen.com/articles/10.1186/s40537-016-0043-6 [Page 10.]
[18] X. Han, Z. Zhang, N. Ding, Y. Gu, X. Liu, Y. Huo, J. Qiu, Y. Yao, A. Zhang, L. Zhang, W. Han, M. Huang, Q. Jin, Y. Lan, Y. Liu, Z. Liu,Z. Lu, X. Qiu, R. Song, J. Tang, J.-R. Wen, J. Yuan, W. X. Zhao, and J. Zhu, “Pre-trained models: Past, present and future,” AI Open, vol. 2, pp. 225–250, 2021. doi: 10.1016/j.aiopen.2021.08.002. [Online]. Available: https://linkinghub.elsevier.com/retrieve/pii/S2666651021000231 [Page 10.]
[19] G. Bonaccorso, Machine Learning Algorithms, 2nd ed. Packt, 2018. ISBN 978-1-78934-799-9 [Page 11.]
[20] M. Buckland and F. Gey, “The relationship between Recall and Precision,” Journal of the American Society for Information Science, vol. 45, no. 1, pp. 12–19, 1994. [Page 11.]
[21] E. Ferrara, P. De Meo, G. Fiumara, and R. Baumgartner, “Web data extraction, applications and techniques: A survey,” Knowledge-Based Systems, vol. 70, pp. 301–323, Nov. 2014. doi: 10.1016/j.knosys.2014.07.007. [Online]. Available: https://linkinghub.elsevier.com/retrieve/pii/S0950705114002640 [Page 12.]
[22] S. M. Selkow, “The tree-to-tree editing problem,” Information Processing Letters, vol. 6, no. 6, pp. 184–186, Dec. 1977. doi: 10.1016/0020-0190(77)90064-3. [Online]. Available: https://linkinghub.elsevier.com/retrieve/pii/0020019077900643 [Page 12.]
[23] P. Kilpeläinen, “Tree matching problems with applications to structured text databases,” Ph.D dissertation, University of Helsinki, Department of Computer Science, Helsinki, Finland, Nov. 1992. [Page 12.]
[24] N. Kushmerick, D. S. Weld, and R. B. Doorenbos, “Wrapper Induction for Information Extraction,” in IJCAI, 1997. [Page 12.]
[25] R. Mooney, “Relational learning of pattern-match rules for information extraction,” in Proceedings of the sixteenth national conference on artificial intelligence, vol. 328, 1999, p. 334. [Page 12.]
[26] S. Soderland, “Learning information extraction rules for semi-structured and free text,” Machine learning, vol. 34, no. 1, pp. 233–272, 1999, publisher: Springer. [Page 12.]
[27] I. Muslea and others, “Extraction patterns for information extraction tasks: A survey,” in The AAAI-99 workshop on machine learning for information extraction, vol. 2. Orlando Florida, 1999, issue: 2. [Page 13.]
[28] S. Zhang and K. Balog, “Web Table Extraction, Retrieval, and Augmentation: A Survey,” ACM Transactions on Intelligent Systems and Technology, vol. 11, no. 2, pp. 1–35, Apr. 2020. doi: 10.1145/3372117. [Online]. Available: https://dl.acm.org/doi/10.1145/3372117 [Page 13.]
[29] J. Devlin, M.-W. Chang, K. Lee, and K. Toutanova, “BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding,” arXiv:1810.04805 [cs], May 2019. [Online]. Available: http://arxiv.org/abs/1810.04805 [Pages 15, 17, and 36.]
[30] Y. Zhu, R. Kiros, R. Zemel, R. Salakhutdinov, R. Urtasun, A. Torralba, and S. Fidler, “Aligning books and movies: Towards story-like visual explanations by watching movies and reading books,” in Proceedings of the IEEE international conference on computer vision, 2015, pp. 19–27.[Page 16.]
[31] A. Wang, A. Singh, J. Michael, F. Hill, O. Levy, and S. R. Bowman, “GLUE: A Multi-Task Benchmark and Analysis Platform for Natural Language Understanding,” arXiv:1804.07461 [cs], Feb. 2019. [Online]. Available: http://arxiv.org/abs/1804.07461 [Page 16.]
[32] P. Rajpurkar, J. Zhang, K. Lopyrev, and P. Liang, “SQuAD: 100,000+Questions for Machine Comprehension of Text,” arXiv:1606.05250 [cs], Oct. 2016. [Online]. Available: http://arxiv.org/abs/1606.05250[Page 16.]
[33] R. Zellers, Y. Bisk, R. Schwartz, and Y. Choi, “SWAG: A LargeScale Adversarial Dataset for Grounded Commonsense Inference,” arXiv:1808.05326 [cs], Aug. 2018. [Online]. Available: http://arxiv.org/abs/1808.05326 [Page 16.]
[34] Y. Zhou, Y. Sheng, N. Vo, N. Edmonds, and S. Tata, “Simplified DOM Trees for Transferable Attribute Extraction from the Web,”arXiv:2101.02415 [cs], Jan. 2021. [Online]. Available: http://arxiv.org/abs/2101.02415 [Pages 16, 18, 19, 22, and 41.]
[35] Q. Hao, R. Cai, Y. Pang, and L. Zhang, “From one tree to a forest: a unified solution for structured web data extraction,” in Proceedings of the34th international ACM SIGIR conference on Research and development in Information - SIGIR ’11. Beijing, China: ACM Press, 2011. doi:10.1145/2009916.2010020. ISBN 978-1-4503-0757-4 p. 775. [Online]. Available: http://portal.acm.org/citation.cfm?doid=2009916.2010020 [Pages 17, 18, 22, and 40.]
[36] J. Pennington, R. Socher, and C. Manning, “Glove: Global Vectors for Word Representation,” in Proceedings of the 2014 Conference on Empirical Methods in Natural Language Processing (EMNLP). Doha, Qatar: Association for Computational Linguistics, 2014. doi: 10.3115/v1/D14-1162 pp. 1532–1543. [Online]. Available: http://aclweb.org/anthology/D14-1162 [Page 17.]
[37] J. Li, Y. Xu, L. Cui, and F. Wei, “MarkupLM: Pre-training of Text and Markup Language for Visually-rich Document Understanding,”arXiv:2110.08518 [cs], Oct. 2021. [Online]. Available: http://arxiv.org/abs/2110.08518 [Pages 17, 22, 27, and 29.]
[38] B. Y. Lin, Y. Sheng, N. Vo, and S. Tata, “FreeDOM: A Transferable Neural Architecture for Structured Information Extraction on Web Documents,” in Proceedings of the 26th ACM SIGKDD International Conference on Knowledge Discovery & Data Mining. Virtual Event CA USA: ACM, Aug. 2020. doi: 10.1145/3394486.3403153. ISBN 978-1-4503-7998-4 pp. 1092–1102. [Online]. Available: https://dl.acm.org/doi/10.1145/3394486.3403153 [Page 18.]
[39] X. Deng, P. Shiralkar, C. Lockard, B. Huang, and H. Sun, “DOMLM: Learning Generalizable Representations for HTML Documents,” arXiv:2201.10608 [cs], Jan. 2022. [Online]. Available: http://arxiv.org/abs/2201.10608 [Page 18.]
[40] K. Peffers, T. Tuunanen, C. E. Gengler, M. Rossi, W. Hui, V. Virtanen, and J. Bragge, “Design Science Research Process: A Model for Producing and Presenting Information Systems Research,” arXiv:2006.02763 [cs], Jun. 2020. [Online]. Available: http://arxiv.org/abs/2006.02763 [Page 21.]
[41] “AWS Deep Learning AMIs.” [Online]. Available: https://aws.amazon.com/machine-learning/amis/ [Page 23.]
[42] “Amazon EC2 G4 Instances.” [Online]. Available: https://aws.amazon.com/ec2/instance-types/g4/ [Page 23.]
[43] “Pytorch.” [Online]. Available: https://pytorch.org/ [Page 27.]
[44] “Transformers.” [Online]. Available: https://huggingface.co/transformers [Page 27.]
[45] “pickle — Python object serialization,” Apr. 2022. [Online]. Available: https://docs.python.org/3/library/pickle.html [Page 27.]
[46] Y. Liu, M. Ott, N. Goyal, J. Du, M. Joshi, D. Chen, O. Levy, M. Lewis, L. Zettlemoyer, and V. Stoyanov, “RoBERTa: A Robustly Optimized BERT Pretraining Approach,” arXiv:1907.11692 [cs], Jul. 2019. [Online]. Available: http://arxiv.org/abs/1907.11692 [Page 29.]
[47] M. Malmsten, L. Börjeson, and C. Haffenden, “Playing with Words at the National Library of Sweden – Making a Swedish BERT,” arXiv:2007.01658 [cs], Jul. 2020. [Online]. Available: http://arxiv.org/abs/2007.01658 [Page 29.]
[48] “The AI community building the future.” [Online]. Available: https://huggingface.co/ [Page 29.]
[49] “Cloud TPU.” [Online]. Available: https://cloud.google.com/tpu/ [Page 36.]
[50] D. M. W. Powers, “Evaluation: from precision, recall and F-measure to ROC, informedness, markedness and correlation,” arXiv:2010.16061 [cs, stat], Oct. 2020. [Online]. Available: http://arxiv.org/abs/2010.16061 [Page 36.]
