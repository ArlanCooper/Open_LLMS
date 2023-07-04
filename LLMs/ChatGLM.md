# ChatGLM

## 地址

- ChatGLM-6B

https://github.com/THUDM/ChatGLM-6B/tree/main

- ChatGLM2-6B


https://github.com/THUDM/ChatGLM2-6B




## 原理

ChatGLM-6B 是一个开源的、支持中英双语的对话语言模型，基于 General Language Model (GLM) 架构，具有 62 亿参数。ChatGLM-6B 使用了和 ChatGPT 相似的技术，针对中文问答和对话进行了优化。经过约 1T 标识符的中英双语训练，辅以监督微调、反馈自助、人类反馈强化学习等技术的加持，62 亿参数的 ChatGLM-6B 已经能生成相当符合人类偏好的回答。

### GLM原理

#### 背景

1、autoregressive自回归模型（AR模型）：代表作GPT。本质上是一个left-to-right的语言模型。通常用于生成式任务，在长文本生成方面取得了巨大的成功，比如自然语言生成（NLG）领域的任务：摘要、翻译或抽象问答。当扩展到十亿级别参数时，表现出了少样本学习能力。缺点是单向注意力机制，在NLU任务中，无法完全捕捉上下文的依赖关系。

2、autoencoding自编码模型（AE模型）：代表作BERT。是通过某个降噪目标（比如MLM）训练的双向文本编码器。编码器会产出适用于NLU任务的上下文表示，但无法直接用于文本生成。

3、encoder-decoder（Seq2seq模型）：代表作T5。采用双向注意力机制，通常用于条件生成任务，比如文本摘要、机器翻译等。

三种预训练框架各有利弊，没有一种框架在以下三种领域的表现最佳：自然语言理解（NLU）、无条件生成以及条件生成。T5曾经尝试使用MTL的方式统一上述框架，然而自编码和自回归目标天然存在差异，简单的融合自然无法继承各个框架的优点。

在这个天下三分的僵持局面下，GLM诞生了。

GLM模型基于autoregressive blank infilling方法，结合了上述三种预训练模型的思想。

#### GLM预训练框架

GLM有什么特点？又是如何将其他框架的优势巧妙融合的呢？

1、自编码思想：在输入文本中，随机删除连续的tokens。

2、自回归思想：顺序重建连续tokens。在使用自回归方式预测缺失tokens时，模型既可以访问corrupted文本，又可以访问之前已经被预测的spans。

3、span shuffling + 二维位置编码技术。

4、通过改变缺失spans的数量和长度，自回归空格填充目标可以为条件生成以及无条件生成任务预训练语言模型。

#### 自回归空格填充任务

给定一个输入文本 $x = [x_1,...,x_n]$，可以采样得到多个文本spans
。为了充分捕捉各spans之间的相互依赖关系，可以对spans的顺序进行随机排列，得到所有可能的排列集合，其中：。所以预训练目标很清晰：

GLM自回归空格填充任务的技术细节：

1、输入x可以被分成两部分：Part A是被损坏的文本$x_{corrupt}$，Part B由masked spans组成。假设原始输入文本是[x1, x2, x3, x4, x5, x6]，采样的两个文本片段是[x3]以及[x5, x6]。那么mask后的文本序列是：x1, x2, [M], x4, [M]，即Part A；同时我们需要对Part B的片段进行shuffle。每个片段使用[S]填充在开头作为输入，使用[E]填充在末尾作为输出。

2、二维位置编码：Transformer使用位置编码来标记tokens中的绝对和相对位置。在GLM中，使用二维位置编码，第一个位置id用来标记Part A中的位置，第二个位置id用来表示跨度内部的相对位置。这两个位置id会通过embedding表被投影为两个向量，最终都会被加入到输入token的embedding表达中。

3、观察GLM中自定义attention mask的设计，非常巧妙：

（1）Part A中的tokens彼此可见，但是不可见B中的任意tokens。

（2）Part B tokens可见Part A。

（3）Part B tokens可见B中过去的tokens，不可见B中未来的tokens。

4、采样方式：文本片段的采样遵循泊松分布，重复采样，直到原始tokens中有15%被mask。

5、总结：模型可以自动学习双向encoder（Part A）以及单向decoder（Part B）。

#### 多目标预训练

上述方法适合于NLU任务。作者希望可以训练一个既可以解决NLU任务，又具备文本生成能力的模型。因此除了空格填充目标之外，还需要增加一个生成长文本目标的任务。具体包含以下两个目标：

1、文档级别。从文档中采样一个文本片段进行mask，且片段长度为文档长度的50%～100%。这个目标用于长文本生成。

2、句子级别。限制被mask的片段必须是完整句子。多个片段需覆盖原始tokens的15%。这个目标是用于预测完整句子或者段落的seq2seq任务。

#### 模型结构

GLM在原始single Transformer的基础上进行了一些修改：

1）重组了LN和残差连接的顺序；

2）使用单个线性层对输出token进行预测；

3）激活函数从ReLU换成了GeLUS。

#### GLM微调

对于下游NLU任务来说，通常会将预训练模型产出的序列或tokens表达作为输入，使用线性分类器预测label。所以预训练与微调之间存在天然不一致。

作者按照PET的方式，将下游NLU任务重新表述为空白填充的生成任务。具体来说，比如给定一个已标注样本(x, y)，将输入的文本x转换成一个包含mask token的完形填空问题。比如，情感分类任务可以表述为："{SENTENCE}. It’s really [MASK]"。输出label y也同样会被映射到完形填空的答案中。“positive” 和 “negative” 对应的标签就是“good” 和 “bad。



# 参考文献

1. [国内开源大模型列表](https://juejin.cn/post/7247089411803562040)
2. [开源模型大模型](https://my.oschina.net/oscpyaqxylk/blog/8727824)
3. [开源大模型地址](https://github.com/eugeneyan/open-llms)
4. [GLM原理](https://zhuanlan.zhihu.com/p/630134021)
5. [GLM原理讲解](https://zhuanlan.zhihu.com/p/637382548)
6. [chatglm情况介绍](https://chatglm.cn/blog)

