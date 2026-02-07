---
title: AI大事件
date: 2026-1-15
categories: AI
tags: 
- AI
---

## 2012 - 2015 年：深度学习大爆炸

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2012.09 | CNN 崛起 (AlexNet) | 深度卷积神经网络首次在 ImageNet 夺冠，准确率远超传统算法 | AlexNet 开启深度学习革命 | [AlexNet Paper (NIPS 2012)](https://proceedings.neurips.cc/paper/2012/file/c399862d3b9d6b76c8436e924a68c45b-Paper.pdf) |
| 2014.06 | GANs (生成对抗网络) | 通过“生成器”与“判别器”博弈，开创生成式 AI 先河 | Ian Goodfellow 提出 GANs 概念 | [GANs Paper (2014)](https://arxiv.org/abs/1406.2661) |
| 2015.06 | YOLO 诞生 (v1) | 提出“You Only Look Once”，将检测视为回归问题，实现实时目标检测 | Joseph Redmon 发布 YOLO，改变了视觉检测效率 | [YOLO v1 Paper](https://arxiv.org/abs/1506.02640) |
| 2015.12 | ResNet (残差网络) | 解决深层网络退化问题，使训练百层甚至千层网络成为可能 | 微软发布 ResNet，斩获 ILSVRC 五项第一 | [ResNet Paper (2015)](https://arxiv.org/abs/1512.03385) |

## 2016 - 2018 年：感知到理解的飞跃

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2016.03 | 强化学习突破 | 深度学习与强化学习结合，处理极端复杂的博弈空间 | AlphaGo 以 4:1 击败人类顶尖棋手李世石 | [DeepMind: AlphaGo](https://www.google.com/deepmind/blog/alphago-the-first-computer-program-to-ever-beat-a-professional-player-at-the-game-of-go/) |
| 2017.06 | **Transformer 诞生** | 抛弃 RNN，采用“注意力机制”并行处理数据，LLM 的核心架构 | Google 发布《Attention Is All You Need》 | [Transformer Paper (2017)](https://arxiv.org/abs/1706.03762) |
| 2018.10 | BERT (预训练双向编码) | 引入“遮罩语言模型”，极大提升 NLP 任务的理解能力 | Google 发布 BERT，刷新 11 项 NLP 纪录 | [BERT Paper (2018)](https://arxiv.org/abs/1810.04805) |

## 2019 - 2021 年：参数爆炸与生成预热

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2020.05 | **GPT-3 (千亿级参数)** | 1750 亿参数，首次展现“上下文学习（In-context Learning）”能力 | OpenAI 发布 GPT-3，AI 开始展现惊人的创作潜力 | [OpenAI: GPT-3 Paper. Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165) |
| 2020.05 | DETR | 抛弃 NMS、锚框等手工设计，首次将 Transformer 引入物体检测 | Facebook (Meta) 发布 Transformer架构的DETR挑战CNN架构的YOLO，开启视觉检测新范式 | [DETR Paper (ECCV 2020)](https://arxiv.org/abs/2005.12872) |
| 2020.10 | ViT (Vision Transformer) | 抛弃卷积网络(CNN)，将图像切块后像文本一样处理 | Google 发布《An Image is Worth 16x16 Words》，视觉架构转向 Transformer，挑战 CNN 霸主地位 | [ViT Paper](https://arxiv.org/abs/2010.11929) |
| 2021.01 | CLIP & DALL-E | 借助 ViT 架构将图文关联，实现多模态理解与生成 | OpenAI 发布 DALL-E，开启多模态生成元年 | [OpenAI Blog: DALL-E](https://openai.com/blog/dall-e/) |
| 2021.06 | GitHub Copilot | 基于 OpenAI Codex 的代码补全，AI 辅助编程商业化 | GitHub 推出 Copilot 预览版 | [GitHub Blog: Introducing Copilot](https://github.blog/2021-06-29-introducing-github-copilot-ai-pair-programmer/) |

## 2022 年：大模型觉醒之年

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
|------|------|--------|--------|------|
| 2022.01 | CoT (思维链) | 通过 "Let's think step by step" 诱导模型输出中间推理步骤 | Google 发布思维链研究，大幅提升模型逻辑推理能力 | [CoT Paper (Google Brain)](https://arxiv.org/abs/2201.11903) |
| 2022.03 | Chinchilla Scaling Laws | 提出参数量与数据量的最佳平衡比例（约 1:20） | DeepMind 发布 Chinchilla，70B 模型击败了 175B 的 GPT-3 | [Chinchilla Paper](https://arxiv.org/abs/2203.15556) |
| 2022.10 | ReAct 框架 | 将“推理(Reason)”与“行动(Act)”结合，允许模型在思考过程中调用搜索等外部工具 | Google & Princeton 发布 ReAct，奠定了 LLM Agent 的底层逻辑 | [ReAct Paper](https://arxiv.org/abs/2210.03629) |
| 2022.11 | ChatGPT 爆发 | 基于 GPT-3.5 的对话 AI，首次让公众体验“类人对话” | OpenAI 发布 ChatGPT，5 天破百万用户 | [OpenAI Blog: Introducing ChatGPT](https://openai.com/blog/chatgpt) |
| 2022.Q4 | Prompt Engineering（提示工程） | 通过精心设计输入指令引导 LLM 输出 | 成为早期 AI 使用者的核心技能 | [Google Developers: Prompt Design Guide](https://cloud.google.com/discover/what-is-prompt-engineering) |

## 2023 年：多模态 + 开源崛起

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
|------|------|--------|--------|------|
| 2023.03 | GPT-4 发布 | 多模态（图像+文本）、更强推理、上下文达 32K | OpenAI 宣称“AGI 重要一步” | [OpenAI GPT-4 Technical Report](https://cdn.openai.com/papers/gpt-4.pdf) |
| 2023.04 | LLM Agent 初现 | LLM 能调用工具、规划任务、自我反思 | AutoGPT、BabyAGI 开源引爆 GitHub | [AutoGPT GitHub](https://github.com/Significant-Gravitas/AutoGPT)<br>[BabyAGI GitHub](https://github.com/yoheinakajima/babyagi) |
| 2023.07 | 开源大模型浪潮 | Meta 开源 Llama，打破闭源垄断 | Llama、Falcon、Mistral 推动本地部署 | [Meta Llama 2 Announcement](https://ai.meta.com/llama/)<br>[Falcon LM (TII)](https://falconllm.tii.ae/)<br>[Mistral AI Launch](https://mistral.ai/news/announcing-mistral-7b/) |
| 2023.09 | RAG（检索增强生成） | 让 LLM 结合私有知识库回答问题 | 成为企业落地 LLM 的首选架构 | [Lewis et al., "Retrieval-Augmented Generation", 2020 (奠基)](https://arxiv.org/abs/2005.11401)<br>[LangChain RAG Docs](https://python.langchain.com/docs/use_cases/question_answering/) |
| 2023.12 | AI Coding 工具普及 | Copilot 全面商用，代码生成进入日常开发 | GitHub Copilot 覆盖超 3 万企业 | [GitHub Copilot Enterprise Launch](https://github.blog/changelog/2023-06-29-copilot-june-2023-update/) |

##  2024 年：Agentic 智能体元年

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
|------|------|--------|--------|------|
| 2024.02 | Multimodal Agents | 能看图、听音、操作 GUI 的智能体 | Google 推出 Astra，OpenAI 展示 GPT-4V 操控手机 | [Google Astra Demo (I/O 2024)](https://deepmind.google/models/project-astra/)<br>[OpenAI GPT-4o Vision Demo](https://openai.com/index/hello-gpt-4o/) |
| 2024.05 | Memory & Reflection | Agent 具备长期记忆与事后复盘能力 | Stanford 发布 “Reflexion” 框架 | [Shinn & Cassano et al., "Reflexion: Language Agents with Verbal Reinforcement Learning", NeurIPS 2023](https://arxiv.org/abs/2303.11366) |
| 2024.08 | Function Calling 2.0 | 更可靠的工具调用协议（如 MCP 前身） | Anthropic、OpenAI 升级 Tool Use API | [Anthropic Tools Documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)<br>[OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) |
| 2024.10 | AI OS / AI Native App | 应用围绕 AI 重构，而非“加个聊天框” | Notion AI、Cursor、Windsurf 等新锐产品崛起 | [Notion AI](https://www.notion.so/product/ai)<br>[Cursor.sh](https://cursor.com/)<br>[Windsurf.ai](https://windsurf.ai/) |

## 2025 年：规范驱动 + 技术融合爆发年

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
|------|------|--------|--------|------|
| 2025.01 | DeepSeek | 中国开源模型进入全球第一梯队 | DeepSeek App和DeepSeek-R1开源模型发布 | [DeepSeek-R1 发布，性能对标 OpenAI o1 正式版](https://api-docs.deepseek.com/zh-cn/news/news250120) |
| 2025.01 | Spec-Driven Development (SDD) | 先写规范（Spec），AI 自动生成并维护代码 | AWS 推出 Kiro，GitHub 推出 Spec-kit | [AWS Kiro Announcement (re:Invent 2024)](https://aws.amazon.com/cn/blogs/china/use-kiro-specification-driven-development-to-accelerate-data-quality-construction/)<br>[GitHub Spec-kit Docs](https://github.com/github/spec-kit) |
| 2025.05 | MCP (Model Communication Protocol) | 统一 LLM 与外部工具通信的标准协议 | 类似“AI 的 USB-C”，被 Cursor、Continue、Claude 采纳 | [MCP Specification (GitHub)](https://github.com/modelcontextprotocol/modelcontextprotocol) |
| 2025.07 | Agentic IDE | IDE 内置自主编程智能体（非仅补全） | Cursor Pro、Trae、Qoder 支持“自然语言建项目” | [Cursor Agentic Mode](https://cursor.com/cn/docs/agent/modes#agent)<br>[Trae.ai](https://trae.ai/)<br>[Alibaba Qoder](https://qoder.com/) |
| 2025.09 | AI Factories / AI DevOps | 用 AI 自动构建、测试、部署其他 AI 应用 | Microsoft 提出“AI 生产 AI”范式 | [Microsoft Build 2025 Keynote](https://news.microsoft.com/build-2025/) |
| 2025.10 | Skills (技能) | 模块化、可共享的 Agent 能力包 | Anthropic 在 Claude Code 中正式支持 Skills | [Introducing Agent Skills](https://claude.com/blog/skills) |


> * [Cursor Changelog](https://cursor.com/cn/changelog)
> * [Claude Code Changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
> * [OpenCode Changelog](https://opencode.ai/changelog)
> * [Kiro Changelog](https://kiro.dev/changelog/)
> * [Qoder Changelog](https://qoder.com/changelog)
> * [Manus更新日志](https://manus.im/zh-cn/updates)
> * [Copilot Changelog](https://github.blog/changelog/label/copilot/)
> * [OpenAI Changelog](https://platform.openai.com/docs/changelog)
> * [Anthropic Changelog](https://platform.claude.com/docs/en/release-notes/overview)
> * [Huggingface 论文排行榜](https://huggingface.co/papers/trending)


## 视觉检测演进专项（2013 - 2024）

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2013.11 | **R-CNN (二阶段检测)** | 开启深度学习检测时代。采用“先选框再分类”的 **CNN** 两阶段法，精度高但计算量巨大（非实时） | Ross Girshick 发布 R-CNN，奠定物体检测基础架构 | [R-CNN Paper](https://arxiv.org/abs/1311.2524) |
| 2015.06 | **YOLO v1 (一阶段检测)** | **对比 R-CNN：** 同样基于 **CNN**，但将检测简化为单一回归问题。牺牲微小精度换取极致速度，实现实时检测 | Joseph Redmon 发布 YOLO，改变了工业界视觉落地进程 | [YOLO v1 Paper](https://arxiv.org/abs/1506.02640) |
| 2015.12 | ResNet (残差网络) | 引入残差连接，解决了深度 **CNN** 训练中的梯度消失问题，成为后来所有 YOLO 版本的强力后盾 | 微软发布 ResNet，斩获 ILSVRC 五项第一 | [ResNet Paper](https://arxiv.org/abs/1512.03385) |
| 2020.04 | YOLOv4 / v5 | 引入 CSPNet 等优化，将 **CNN** 架构的检测性能榨干到极致 | AlexeyAB 与 Ultralytics 发布，成为全球部署最广的检测工具 | [YOLOv5 GitHub](https://github.com/ultralytics/yolov5) |
| 2020.05 | **DETR (视觉 Transformer)** | **架构革命：** 彻底抛弃 CNN 时代的锚框和 NMS 后处理，首次将 Transformer 引入检测任务 | Facebook 发布 DETR，开启视觉检测“去卷积”进程 | [DETR Paper](https://arxiv.org/abs/2005.12872) |
| 2022.03 | ViT-Adapter | 将 Transformer 的全局建模能力引入检测主干网络，在大尺寸图像检测上超越传统 **CNN** | 视觉架构正式开始从卷积向注意力机制大迁移 | [ViT-Adapter Paper](https://arxiv.org/abs/2205.08534) |
| 2024.03 | **RT-DETR (实时 Transformer)** | **地位更替：** 解决了 Transformer 速度慢的顽疾。在相同延迟下精度全面超越 YOLOv8 | 百度发布 RT-DETR，标志着 Transformer 在实时赛道击败 **CNN** | [RT-DETR Paper](https://arxiv.org/abs/2304.08069) |
| 2024.Q2 | Grounding DINO | 结合大语言模型，通过文字指令实现“零样本”物体检测 | 物体检测从单一视觉识别进化为多模态语义理解 | [Grounding DINO](https://github.com/IDEA-Research/GroundingDINO) |

## NLP (自然语言处理) 专项

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2013.01 | Word2Vec (词向量) | 将单词转化为稠密向量，通过数学距离表示语义相似度 | Google 发布 Word2Vec，开启 NLP “词表征”时代 | [Word2Vec Paper](https://arxiv.org/abs/1301.3781) |
| 2014.09 | Seq2Seq + Attention | 引入编码器-解码器架构与注意力机制，解决变长序列处理难题 | Bahdanau 等提出 Attention，奠定翻译任务基础 | [Attention Paper (2014)](https://arxiv.org/abs/1409.0473) |
| 2017.06 | **Transformer 架构** | **架构分水岭：** 彻底抛弃 RNN/CNN，利用自注意力机制实现大规模并行训练 | Google 发布《Attention Is All You Need》 | [Transformer Paper](https://arxiv.org/abs/1706.03762) |
| 2018.10 | BERT (双向预训练) | 通过遮罩语言模型（MLM）获取上下文语义，统治理解类任务 | Google 发布 BERT，刷新 11 项 NLP 纪录 | [BERT Paper (2018)](https://arxiv.org/abs/1810.04805) |
| 2020.05 | GPT-3 (千亿级参数) | 首次展现“上下文学习”能力，证明了 Scaling Law 的巨大潜力 | OpenAI 发布 GPT-3，开启大模型（LLM）狂潮 | [GPT-3 Paper](https://arxiv.org/abs/2005.14165) |
| 2022.01 | CoT (思维链) | 通过中间推理步骤引导模型，AI 从“预测概率”转向“逻辑模拟” | Google Brain 发布思维链研究，攻克复杂数学题 | [CoT Paper](https://arxiv.org/abs/2201.11903) |
| 2022.11 | ChatGPT (RLHF) | 引入人类反馈强化学习，让模型生成内容符合人类偏好与安全规范 | OpenAI 发布 ChatGPT，解决“对齐”问题 | [OpenAI Blog: ChatGPT](https://openai.com/blog/chatgpt) |
| 2024.09 | **OpenAI o1 (推理模型)** | **范式演进：** 通过强化学习诱导模型在输出前进行长时间的自我推理 | OpenAI 发布 o1 预览版，显著提升理科逻辑能力 | [OpenAI o1 Announcement](https://openai.com/index/introducing-openai-o1-preview/) |
| 2025.01 | **DeepSeek-R1 (推理开源)** | 纯强化学习训练的开源推理模型，低成本复现 o1 级别性能 | DeepSeek 发布 R1 系列，打破推理大模型闭源垄断 | [DeepSeek-R1 News](https://api-docs.deepseek.com/zh-cn/news/news250120) |
| 2025.Q4 | Logic-Native LLMs | 逻辑推理层与语言表述层彻底分离，解决大模型幻觉问题 | 工业界普及“逻辑内核”架构，模型回答准确率趋近 100% | [LLM Logic Research](%23) |

## 语音处理 (Speech) 专项

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2012.11 | DNN-HMM (深度神经网络) | DNN 取代传统的混合高斯模型（GMM），识别率实现质的飞跃 | 微软与 Google 联合宣布深度学习在语音识别的突破 | [DNN-HMM Research](https://ieeexplore.ieee.org/document/6296526) |
| 2016.09 | **WaveNet (神经生成)** | 摒弃拼接合成，基于神经网络逐个采样点生成原始音频波形 | DeepMind 发布 WaveNet，让机器合成音接近人声 | [WaveNet Paper](https://arxiv.org/abs/1609.03499) |
| 2017.12 | Tacotron 2 | 简化 TTS 流程，实现从字符到梅尔频谱的端到端合成 | Google 发布 Tacotron 2，确立了现代 TTS 的基本范式 | [Tacotron 2 Paper](https://arxiv.org/abs/1712.05884) |
| 2022.09 | **Whisper (通用识别)** | 基于 Transformer 的大规模弱监督预训练，解决杂音与多语言难题 | OpenAI 开源 Whisper 语音识别系列模型 | [Whisper Paper](https://arxiv.org/abs/2212.04356) |
| 2023.01 | VALL-E (神经编解码) | 基于离散代码的神经编解码语言模型，实现“3秒克隆”声音 | 微软发布 VALL-E，开启了个性化语音生成元年 | [VALL-E Paper](https://arxiv.org/abs/2301.02111) |
| 2024.05 | **GPT-4o (原生语音交互)** | **范式演进：** 彻底抛弃 ASR+TTS 链路，实现音频输入输出的端到端训练 | OpenAI 发布 GPT-4o，延迟低至 320ms，具备情感表达 | [OpenAI Blog: GPT-4o](https://openai.com/index/hello-gpt-4o/) |
| 2025.01 | Audio-Reasoning (语音推理) | 语音模型具备“思考”能力，能通过音调、语气推断用户真实意图 | OpenAI 升级 Advanced Voice Mode 推理能力 | [OpenAI Voice Updates](%23) |
| 2025.10 | Skill-Based Voice Agents | 将语音交互与 Agent 技能包结合，AI 可通过语音操控外部应用 | Anthropic 在 Claude Code 语音版中支持 Skills 调用 | [Introducing Voice Skills](https://claude.com/blog/skills) |

### 多模态生成 (AIGC) 专项：从像素重组到物理模拟

该领域完成了从“乱涂乱画”到“理解物理世界规律”的跨越。

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2014.06 | GANs (对抗网络) | 生成器与判别器博弈，开启了深度学习生成图像的先河 | Ian Goodfellow 提出 GANs 架构 | [GANs Paper](https://arxiv.org/abs/1406.2661) |
| 2021.12 | **Diffusion (扩散模型)** | 通过“去噪”过程生成图像，稳定性与多样性全面超越 GANs | OpenAI 发布 GLIDE，Stable Diffusion 随后爆发 | [Diffusion Paper](https://arxiv.org/abs/2112.10741) |
| 2022.08 | Stable Diffusion | 开源图像生成模型，支持通过 Prompt 精确控图 | Stability AI 发布 SD v1.4，引爆 AI 绘画狂潮 | [SD Launch](https://stability.ai/blog/stable-diffusion-public-release) |
| 2024.02 | **Sora (视频生成)** | 基于 Transformer 处理时空切块，生成长达 1 分钟的一致性视频 | OpenAI 发布 Sora，展现了“模拟物理世界”的潜力 | [Sora Blog](https://openai.com/sora) |
| 2025.12 | **World Models (世界模型)** | AI 不仅生成画面，还能预测物体碰撞、重力等物理反馈 | 视觉生成模型与物理引擎彻底融合，用于机器人预训练 | [World Models Research](https://worldmodelresearch.com/) |

### 具身智能 (Embodied AI) 专项：AI 走进现实世界

这是 AI 的“终极战场”，让算法拥有实体，在物理空间执行任务。

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2022.08 | PaLM-SayCan | 将大语言模型作为机器人的“大脑”，规划复杂指令 | Google 展示机器人根据指令拿取零食 | [SayCan Paper](https://arxiv.org/abs/2204.01691) |
| 2023.07 | RT-2 (视觉-语言-动作) | 提出 VLA 模型，将视觉识别与机器人动作控制统一训练 | Google 发布首个视觉-语言-动作大模型 | [RT-2 Blog](https://deepmind.google/blog/rt-2-new-model-translates-vision-and-language-into-action/) |
| 2024.03 | **Figure 01 + OpenAI** | 机器人接入大模型，实现边说话边根据视觉反馈整理餐具 | Figure 发布接入 OpenAI 的机器人演示视频 | [Figure AI News](https://www.figure.ai/) |
| 2025.02 | **End-to-End Robotics** | 抛弃手工写代码控制关节，实现“视觉输入-动作输出”的全端到端训练 | 特斯拉 Optimus 实现高度类人的灵巧手操作 | [Tesla AI Day 2025](https://www.tesla.com/AI) |
| 2026.01 | Robot-Brain 标准化 | 类似电脑系统的统一机器人底层系统出现，实现技能跨硬件迁移 | 宇树、Figure 等厂商达成机器人通用指令集共识 | [Unified Robot OS](%23) |

### AI for Science 专项：AI 改变科研范式

AI 开始解决人类几十年无法攻克的科学难题（生物、材料、气象）。

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2020.11 | **AlphaFold 2** | 破解困扰生物学 50 年的“蛋白质折叠”难题 | DeepMind 预测了几乎所有人类已知蛋白质结构 | [AlphaFold 2 Nature](https://www.nature.com/articles/s41586-021-03819-2) |
| 2023.03 | GraphCast | 基于图神经网络的气象预测，精度与速度远超传统数值模拟 | Google 发布全球最准的中期天气预报模型 | [GraphCast Paper](https://arxiv.org/abs/2212.12794) |
| 2023.11 | GNoME (新材料预测) | AI 预测了 220 万种新晶体结构，相当于人类 800 年的知识积累 | DeepMind 发布新材料预测成果 | [GNoME Blog](https://deepmind.google/blog/millions-of-new-materials-discovered-with-deep-learning/) |
| 2025.05 | **Drug-Discovery LLM** | 专用大模型实现从靶点发现到药物分子设计的自动化 | 首款由 AI 全流程设计的抗癌药物进入三期临床 | [AI Medicine Review](https://arxiv.org/html/2409.04481v1) |

## 音乐生成 (Music AI) 专项演进史

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2016.09 | Sony Flow Machines | AI 辅助创作。通过算法学习曲风，辅助人类写出旋律 | 索尼发布全球首支 AI 创作流行曲《Daddy's Car》 | [Sony AI Music](https://www.sonycsl.co.jp/tokyo/2911/) |
| 2019.04 | MuseNet (OpenAI) | 深度学习作曲。能模拟 10 种乐器，结合从古典到流行的曲风 | OpenAI 发布 MuseNet，展示跨流派编排能力 | [MuseNet Blog](https://openai.com/blog/musenet/) |
| 2023.01 | **MusicLM (Google)** | **语义里程碑：** 首次实现根据复杂文本描述（如“带有热带风情的爵士乐”）生成高保真音频 | Google 发布 MusicLM，将音乐生成引入大模型时代 | [MusicLM Paper](https://arxiv.org/abs/2301.11325) |
| 2023.06 | AudioCraft / MusicGen | Meta 开源的音乐生成模型，支持通过旋律和文字共同控制 | Meta 开源 AudioCraft，推动了社区二次开发热潮 | [AudioCraft GitHub](https://github.com/facebookresearch/audiocraft) |
| 2023.12 | **Suno v2** | **全流程集成：** 首次在网页端实现“歌词+旋律+人声”一键生成，降低了创作门槛 | Suno 接入 Copilot 插件，开启 AI 音乐平民化元年 | [Suno v2 Launch](https://suno.com/) |
| **2024.03** | **Suno v3 / Udio** | **工业革命：** 生成质量达到“广播级”，具备复杂的转调、和声及情感表达 | Suno v3 发布；随后 Udio 带着更高的音频保真度横空出世 | [Suno v3 News](https://suno.com/blog/v3) |
| 2025.01 | **Mureka / Suno v5** | **本地化与推理：** 深度理解特定语言（如中文）的曲风韵律，支持更长的生成时长（8分钟+） | 昆仑万维发布 Mureka；Suno 更新至 v5，音质实现无损化 | [Mureka AI](https://www.mureka.ai/) |
| 2026.01 | **Interactive DAW Agent** | AI 进驻数字音频工作站（DAW），支持对生成的音轨进行分层精修 | AI 音乐生成从“一键开盲盒”进化为“可交互生产力” | [AI DAW News](%23) |

### 智能体 (Agentic Workflow) 专项：从工具到员工

不仅是对话，而是 AI 能够自主规划、使用工具、完成闭环任务。

| 时间 | 概念 | 核心内容 | 代表事件 | 链接 |
| --- | --- | --- | --- | --- |
| 2023.04 | AutoGPT / BabyAGI | 首次展示 AI 能够自我拆解任务并循环执行 | 开源社区掀起“自主智能体”热潮 | [AutoGPT GitHub](https://github.com/Significant-Gravitas/AutoGPT) |
| 2024.08 | **MCP (模型通信协议)** | 统一 AI 模型与外部数据库、工具、App 之间的通信标准 | Anthropic 发布 MCP，打破智能体连接壁垒 | [MCP GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol) |
| 2025.01 | **Agentic IDE** | AI 不仅写代码，还能自主运行测试、修复 Bug、部署服务 | Cursor、Windsurf 成为开发者标配 | [Cursor.com](https://cursor.com/) |
| 2025.07 | **Computer Use** | AI 具备操控操作系统（点击鼠标、输入文字）的视觉导航能力 | Anthropic 与 Apple 合作推出系统级控制 Agent | [Claude Blog](https://www.anthropic.com/news/3-5-models-and-computer-use) |
| 2025.10 | Claude Code / Skills | 智能体具备了模块化的“技能包”，能自主完成复杂的软件工程链路 | Anthropic 发布终端原生 AI 工程师 | [Claude Code Release](https://claude.com/blog/skills) |
| 2025.12 | OpenClaw (🦞) | 里程碑：跨OS、跨平台的开源 AI 助手，实现“Any OS, Any Platform”的全局操控 | **OpenClaw** 开源，标志着“个人 AI 雇员”时代的平民化 | [OpenClaw Website](https://openclaw.ai/) |
| 2026.01 | **Agentic OS** | 操作系统围绕 Agent 重新设计，App 演变为 Agent 可调用的组件 |  | Future of OS |


