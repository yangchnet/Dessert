---
author: "李昌"
title: "拥抱大模型（五）：Memory，让大模型拥有记忆"
date: "2024-02-04"
tags: ["memory"]
categories: ["LLM"]
ShowToc: true
TocOpen: true
---

要让大模型有记忆也很简单，你只需要把之前的对话传递给他就行，在ConversationChain中，就使用了这一技巧：
```
from langchain import OpenAI
from langchain.chains import ConversationChain

# 初始化大语言模型
llm = OpenAI(
    temperature=0.5,
    model_name="text-davinci-003"
)

# 初始化对话链
conv_chain = ConversationChain(llm=llm)

# 打印对话的模板
print(conv_chain.prompt.template)
```
来看下模板：
```
The following is a friendly conversation between a human and an AI. The AI is talkative and provides lots of specific details from its context. If the AI does not know the answer to a question, it truthfully says it does not know.

Current conversation:
{history}
Human: {input}
AI:
```
可以看到，在模板里又一个history字段，这里存储了之前对话的上下文信息。当有了 {history} 参数，以及 Human 和 AI 这两个前缀，我们就能够把历史对话信息存储在提示模板中，并作为新的提示内容在新一轮的对话过程中传递给模型。—— 这就是记忆机制的原理。
<a name="vi4XK"></a>
# langchain中提供的记忆机制
<a name="jwOuH"></a>
## ConversationBufferMemory
在 LangChain 中，通过 ConversationBufferMemory（缓冲记忆）可以实现最简单的记忆机制。下面，我们就在对话链中引入 ConversationBufferMemory。
```python
from langchain import OpenAI
from langchain.chains import ConversationChain
from langchain.chains.conversation.memory import ConversationBufferMemory

# 初始化大语言模型
llm = OpenAI(
    temperature=0.5,
    model_name="text-davinci-003")

# 初始化对话链
conversation = ConversationChain(
    llm=llm,
    memory=ConversationBufferMemory()
)

# 第一天的对话
# 回合1
conversation("我姐姐明天要过生日，我需要一束生日花束。")
print("第一次对话后的记忆:", conversation.memory.buffer)
```
输出：
```python
第一次对话后的记忆: 
Human: 我姐姐明天要过生日，我需要一束生日花束。
AI:  哦，你姐姐明天要过生日，那太棒了！我可以帮你推荐一些生日花束，你想要什么样的？我知道有很多种，比如玫瑰、康乃馨、郁金香等等。
```
在下一轮对话中，这些记忆会作为一部分传入提示。
```python
# 回合2
conversation("她喜欢粉色玫瑰，颜色是粉色的。")
print("第二次对话后的记忆:", conversation.memory.buffer)
```
有了记忆机制，LLM 能够了解之前的对话内容，这样简单直接地存储所有内容为 LLM 提供了最大量的信息，但是新输入中也包含了更多的 Token（所有的聊天历史记录），这意味着响应时间变慢和更高的成本。

为了节省token，langchain给我们提供了一系列的机制
<a name="w62EA"></a>
## ConversationBufferWindowMemory缓冲记忆
这个Memory的实现原理类似滑动窗口，它只记录了一定数量的过去互动
```python
from langchain import OpenAI
from langchain.chains import ConversationChain
from langchain.chains.conversation.memory import ConversationBufferWindowMemory

# 创建大语言模型实例
llm = OpenAI(
    temperature=0.5,
    model_name="text-davinci-003")

# 初始化对话链
conversation = ConversationChain(
    llm=llm,
    memory=ConversationBufferWindowMemory(k=1)
)

# 第一天的对话
# 回合1
result = conversation("我姐姐明天要过生日，我需要一束生日花束。")
print(result) # {'input': '我姐姐明天要过生日，我需要一束生日花束。', 'history': '', 'response': ' 哦，你姐姐明天要过生日！那太棒了！你想要一束什么样的花束呢？有很多种类可以选择，比如玫瑰花束、康乃馨花束、郁金香花束等等，你有什么喜欢的吗？'}
# 回合2
result = conversation("她喜欢粉色玫瑰，颜色是粉色的。")
# print("\n第二次对话后的记忆:\n", conversation.memory.buffer)
print(result) # {'input': '她喜欢粉色玫瑰，颜色是粉色的。', 'history': 'Human: 我姐姐明天要过生日，我需要一束生日花束。\nAI: 哦，你姐姐明天要过生日！那太棒了！你想要一束什么样的花束呢？有很多种类可以选择，比如玫瑰花束、康乃馨花束、郁金香花束等等，你有什么喜欢的吗？', 'response': ' 好的，那粉色玫瑰花束怎么样？我可以帮你找到一束非常漂亮的粉色玫瑰花束，你觉得怎么样？'}

# 第二天的对话
# 回合3
result = conversation("我又来了，还记得我昨天为什么要来买花吗？")
print(result) # {'input': '我又来了，还记得我昨天为什么要来买花吗？', 'history': 'Human: 她喜欢粉色玫瑰，颜色是粉色的。\nAI: 好的，那粉色玫瑰花束怎么样？我可以帮你找到一束非常漂亮的粉色玫瑰花束，你觉得怎么样？', 'response': ' 当然记得，你昨天来买花是为了给你喜欢的人送一束粉色玫瑰花束，表达你对TA的爱意。'}
```
这里设置k=1，也就是只会记住与 AI 之间的最新的互动，即只保留上一次的人类回应和 AI 的回应。

这里第二次还能记住第一次的互动，但第三次显然没记住，回答错误。

这种方法不适合记住遥远的互动，但它非常擅长限制使用的 Token 数量。如果只需要记住最近的互动，缓冲窗口记忆是一个很好的选择。但是，如果需要混合远期和近期的互动信息，则还有其他选择。
<a name="eYZlA"></a>
## ConversationSummaryMemory对话总结记忆
ConversationSummaryMemory（对话总结记忆）的思路就是将对话历史进行汇总，然后再传递给 {history} 参数。这种方法旨在通过对之前的对话进行汇总来避免过度使用 Token。<br />ConversationSummaryMemory 有这么几个核心特点。

- 汇总对话：此方法不是保存整个对话历史，而是每次新的互动发生时对其进行汇总，然后将其添加到之前所有互动的“运行汇总”中。
- 使用 LLM 进行汇总：该汇总功能由另一个 LLM 驱动，这意味着对话的汇总实际上是由 AI 自己进行的。
- 适合长对话：对于长对话，此方法的优势尤为明显。虽然最初使用的 Token 数量较多，但随着对话的进展，汇总方法的增长速度会减慢。与此同时，常规的缓冲内存模型会继续线性增长。

使用方法：
```python
from langchain.chains.conversation.memory import ConversationSummaryMemory

# 初始化对话链
conversation = ConversationChain(
    llm=llm,
    memory=ConversationSummaryMemory(llm=llm)
)
```
ConversationSummaryMemory使用的“history”不再是之前人类和 AI 对话的简单复制粘贴，而是经过了总结和整理之后的一个综述信息。

ConversationSummaryMemory 的优点是对于长对话，可以减少使用的 Token 数量，因此可以记录更多轮的对话信息，使用起来也直观易懂。不过，它的缺点是，对于较短的对话，可能会导致更高的 Token 使用。另外，对话历史的记忆完全依赖于中间汇总 LLM 的能力，还需要为汇总 LLM 使用 Token，这增加了成本，且并不限制对话长度。

通过对话历史的汇总来优化和管理 Token 的使用，ConversationSummaryMemory 为那些预期会有多轮的、长时间对话的场景提供了一种很好的方法。然而，这种方法仍然受到 Token 数量的限制。在一段时间后，我们仍然会超过大模型的上下文窗口限制。

<a name="GMnFC"></a>
## ConversationSummaryBufferMemory 对话总结缓冲记忆
它是一种混合记忆模型，结合了上述各种记忆机制，包括 ConversationSummaryMemory 和 ConversationBufferWindowMemory 的特点。这种模型旨在在对话中总结早期的互动，同时尽量保留最近互动中的原始内容。<br />它是通过 max_token_limit 这个参数做到这一点的。当最新的对话文字长度在 300 字之内的时候，LangChain 会记忆原始对话内容；当对话文字超出了这个参数的长度，那么模型就会把所有超过预设长度的内容进行总结，以节省 Token 数量。
```python
from langchain.chains.conversation.memory import ConversationSummaryBufferMemory

# 初始化对话链
conversation = ConversationChain(
    llm=llm,
    memory=ConversationSummaryBufferMemory(
        llm=llm,
        max_token_limit=300))
```
ConversationSummaryBufferMemory 的优势是通过总结可以回忆起较早的互动，而且有缓冲区确保我们不会错过最近的互动信息。当然，对于较短的对话，ConversationSummaryBufferMemory 也会增加 Token 数量。<br />总体来说，ConversationSummaryBufferMemory 为我们提供了大量的灵活性。它是我们迄今为止的唯一记忆类型，可以回忆起较早的互动并完整地存储最近的互动。在节省 Token 数量方面，ConversationSummaryBufferMemory 与其他方法相比，也具有竞争力。
<a name="W3PQP"></a>
## 四种记忆机制的总结
![image.png](https://cdn.nlark.com/yuque/0/2024/png/12695724/1704616489271-6dcabda7-23c0-465b-9e43-c2a83ecb86d8.png#averageHue=%23e0eaf3&clientId=u99794b5d-8792-4&from=paste&height=512&id=u840e0bc5&originHeight=640&originWidth=1660&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=354504&status=done&style=none&taskId=u7a9c20e1-1df7-4e62-8d58-18d4d29d3dd&title=&width=1328)<br />当对话轮次逐渐增加时，各种记忆机制对 Token 的消耗数量：<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/12695724/1704616499375-1dd76f22-f5a7-4cc0-b4bd-56c5dab5b445.png#averageHue=%23fcfbfa&clientId=u99794b5d-8792-4&from=paste&height=1182&id=u221fd1f7&originHeight=1478&originWidth=3676&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=410492&status=done&style=none&taskId=u8c3aa340-cc82-48d6-a8cb-f1600e501f0&title=&width=2940.8)
