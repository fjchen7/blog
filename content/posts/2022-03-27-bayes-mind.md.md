---
title: "贝叶斯思维看世界"
date: 2022-03-27
categories: ["随笔"]
tags: ["思维", "哲学"]
summary: "贝叶斯思维是人脑认识世界的默认方式。"
---

在逻辑学中，「归纳法」和「演绎法」是人类认识世界的两种基本方式。「归纳法」是通过观察若干事例来得出一般性结论，是从**部分推断整体**的过程。比如，如果我观察到一枚硬币掷了10次都出现反面，那我会认为它的质量有问题，猜它下一次还是反面；而「演绎法」相反，需要进行严格的逻辑推理，才能得出结论。还是以硬币为例，我需要测量硬币质量和重心，通过一系列数学公式，得出这枚硬币投掷后必然会是反面的结论。「归纳法」和「演绎法」的不同，是使用**经验**或**逻辑**认识世界的差异。

贝叶斯思维是以**经验**结合**概率**的方式来认识世界的过程，属于「归纳法」的一种。事实上，除非是考试或研究，生活中大部分人都以经验来认识世界。贝叶斯思维很符合人类认识世界的模式。在各种决策和推断里，人脑实际上在无意识地做着贝叶斯计算。

用贝叶斯思维认识事物的过程大致是这样：

1. 每个人都有自己的经验。这个经验是你从出生到现在所接收的所有资讯的总结，包括了书本上学到的知识、亲眼经历、社交媒体浏览的内容、朋友间的聊天等等。
2. 现在发生了一件事，要探究造成它的原因是什么。假设只有$B_1$ $B_2$ $B_3$三种可能的原因。
3. 我们的脑瓜子会潜意识地做贝叶斯计算，对$B_1$ $B_2$ $B_3$都做一个概率计算。假设会有10%是$B_1$，50%是$B_2$，40%会是$B_3$。很自然地，我们会选择概率最高的$B_2$作为我们的判断。
3. 并且，发生的这件事会成为新的信息，更新到我们的经验中。我们会用这份新的经验来继续认识世界，周而复始。

贝叶斯思维是一种不断「进化」的过程：经验可能是错误的，但通过不断地更新经验，你会逐渐接近真实，基于此所做的判断会越来越准确。

## 贝叶斯公式

贝叶斯思维基于贝叶斯公式，最初应用在概率论上。公式如下：

$$
P(B_i|A)=\frac{P(A|B_i)P(B_i)}{\sum\limits_jP(B_j)P(A|B_j)}
$$

$B_i$是事件可能的走向，$A$是发生的事件。$P(B_i|A)$是条件概率，表示在$A$发生后，$B_i$发生的概率。而$P(B_i)$表示经验，称为**先验概率**。

在认识世界的贝叶斯思维里，这个公式体现为：最初我们只有经验$P(B_i)$，现在观察到事件$A$发生后，想要知道事件发生的原因。$P(B_i|A)$是不同原因可能的概率，我们要做的，是计算并从中找出概率最大的一个，作为我们的判断。

在贝叶斯公式里，「进化」的特点体现为$P(B_i)$的更新。这一次计算用的$P(B_i)$，实际上是上次计算的结果$P(B_i|A)$。而这一次得到的$P(B_i|A)$，也会成为下次计算的$P(B_i)$。经验的更新，在这里是一层套一层的迭代过程。

## 例子

在《狼来了》的故事里，村民第一次听到牧童喊狼来了，就急冲冲地拿着锄头和镰刀往山上跑。但频繁上当几次后，在狼真的来了后，牧童大喊大叫，却没人回应，发生悲剧。为什么最后村民不相信牧童？我们可以说，是因为牧童说谎成瘾，所以村民不相信他。但说谎和不相信之间实际上还藏着一层关系：因为说谎成瘾，所以这一次也很有可能是谎言，因此村民不相信他。

这是很典型的贝叶斯思维：「**说谎成瘾**」是一种对过往经验的总结，而「**很有可能是谎言**」是在概率上做了判断，而最后村民选择了概率最大的情况作为行动的依据。

但这个过程是递进式的。假设牧童在喊第3次时，村民才不赶过来。那为什么前两次都赶来了，就第3次不来。这是一个信心递减的过程。直观来理解：

- 第1次：村民不了解牧童，但认为孩子不会说谎，所以选择相信。转换成数字，村民认为真话的概率是100%。
- 第2次：村民对牧童有了不好印象，但还不至于太坏，所以仍然相信这是真话。这里不妨设真话的概率为60%。
- 第3次：有了前两次的前车之鉴，村民认为真话的概率低于谎话，概率变成30%。牧童更有可能说谎，所以村民不相信牧童。

每一次牧童大喊，村民都会根据事实，重新计算自己心中的概率值。而每次的计算，都是基于前面的所有经历。对感官接收到的信息，大脑能无意识地使用贝叶斯计算，这无时不刻在生活中发生着。某种程度上看，贝叶斯思维是人类演化而来的能力，是自然选择的结果。



## 贝叶斯哲学

统计学在长期以来都分为两个派别——频率派和贝叶斯派，双方对概率有着不同的定义。频率派认为，频率是事件在长时间内发生的频率（也称为古典频率），这是事件的固有属性；而贝叶斯派认为，频率是人类对事件发生的信心，是一个主观值。

对概率定义的不同导致了两派哲学思想上的差异。频率派的频率，是客观的「真理」，他们试图中从随机性中得到确定性；而贝叶斯派的频率是「信心」，是主观的，每个人对同一件事的信心可以不同，也不存在唯一的信心值。频率派计算频率，是追求真理的过程。这个过程必定是曲折的，很难达到的。而贝叶斯派追求实用性，他们不关心结论是否为真理或客观事实，他们只关心是否能自圆其说，适用当下的场景。

这种追求真理和追求实用的不同，可以体现在生活里的方方面面。比如，在编程时要使用一个没用过的第三方库，有两种路线

- 从头到尾看一遍官方文档，完整了解原理后，再实现需求。这是频率派的哲学，追求真理。
- 自己摸索或上stackoverflow搜索，直接满足需求。这个过程中可能需要多次折返修改等。这是贝叶斯派的哲学，追求实用。

我想大部分程序员会先选择贝叶斯哲学，先看例子满足需求，而不是先阅读文档。只有在需要做优化或者找不到足够好的例子时，再来研读文档。在节约时间和精力上，这确实是更好的策略。

上面这个例子中，有文档可以阅读，因此「真理」可以追求到。但在现实中的很多场景里，「真理」往往很难甚至是追寻不到。比如，想知道自己的爱好，是很难通过了解自己的个性和能力知道的（认识自己是很难的一件事）。取而代之，我们可以多尝试不同的事情，每次尝试都会加深地你对自身喜好的认识。

## 结语

在贝叶斯主义者眼中，真理是不存在的。任何知识都不是真理，它们的区别无非在于适用性。因此，贝叶斯主义的“至圣先师”乔治·博克斯说过：“所有模型都是错的，有些模型很有用。”这在侧面反映了人类认识世界的局限性——我们或许根本就无法接近真理，只能得到能足够描述当下的知识。但谁又能保证我们所见的世界是真实。

但乐观一点看，随着时间维度的增加，我们收集的信息越多，将会越来越接近真实。这或许是唯一令我们宽慰的地方了。

**我们只能得到部分真相，但可以通过不断收集证据来完善我们的观点，逐渐逼近真相**。