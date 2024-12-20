---
author: "李昌"
title: "拥抱大模型（二）：Prompt，给大模型有用的提示"
date: "2024-02-04"
tags: ["prompt engine"]
categories: ["LLM"]
ShowToc: true
TocOpen: true
---

> 极客时间《LangChain实战课》学习笔记

## 构建prompt的原则
原则（吴恩达版）

- 写出清晰而具体的提示
- 给模型思考的时间

原则（OpenAI版）

- 写清晰的指示
- 给模型提供参考（也就是示例）
- 将复杂任务拆分成子任务
- 给 GPT 时间思考
- 使用外部工具
- 反复迭代问题
<a name="uFTo8"></a>
## prompt的基本结构
![20241220155210](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20241220155210.png)

1. instruction（指令）：告诉大模型要做什么，一个常见且有效的例子是，告诉大模型“你是一个XX专家”
2. context（上下文）：充当模型的额外知识来源，这些知识可以从矢量数据库中得来或通过其他方式拉入
3. prompt input （提示输入）：具体的问题或大模型做的具体事情
4. output indicator（标记要生成的文本的开始）：用一个明显的提示词让大模型开始回答，这一部分不是必须的

**使用langchain构建prompt**
```python
from langchain import PromptTemplate

template = """\
你是业务咨询顾问。
你给一个销售{product}的电商公司，起一个好的名字？
"""
prompt = PromptTemplate.from_template(template)

print(prompt.format(product="鲜花"))
```
```python
prompt = PromptTemplate(
    input_variables=["product", "market"], 
    template="你是业务咨询顾问。对于一个面向{market}市场的，专注于销售{product}的公司，你会推荐哪个名字？"
)
print(prompt.format(product="鲜花", market="高端"))
```
二者效果相同<br />**构建chat prompt**<br />对于像ChatGPT这种聊天模型，langchain提供了ChatPromptTemplate，其中有多种角色类型:
```python
import openai
openai.ChatCompletion.create(
  model="gpt-3.5-turbo",
  messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Who won the world series in 2020?"},
        {"role": "assistant", "content": "The Los Angeles Dodgers won the World Series in 2020."},
        {"role": "user", "content": "Where was it played?"}
    ]
)
```

- system（系统消息，可选）：设置AI的行为，例如，你是一个客服
- user（用户消息）：人类提出的问题
- assistant（助理消息）：可以存储AI以前给出的响应，或在此处给出示例
```python
# 导入聊天消息类模板
from langchain.prompts import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)
# 模板的构建
template="你是一位专业顾问，负责为专注于{product}的公司起名。"
system_message_prompt = SystemMessagePromptTemplate.from_template(template)
human_template="公司主打产品是{product_detail}。"
human_message_prompt = HumanMessagePromptTemplate.from_template(human_template)
prompt_template = ChatPromptTemplate.from_messages([system_message_prompt, human_message_prompt])

# 格式化提示消息生成提示
prompt = prompt_template.format_prompt(product="鲜花装饰", product_detail="创新的鲜花设计。").to_messages()

# 下面调用模型，把提示传入模型，生成结果
import os
os.environ["OPENAI_API_KEY"] = '你的OpenAI Key'
from langchain.chat_models import ChatOpenAI
chat = ChatOpenAI()
result = chat(prompt)
print(result)
```

<a name="AAKNz"></a>
## 给LLM提供样本
根据给予模型样本的多少，可粗略分为Few-Shot ，One-Shot，Zero-Shot<br />在 Few-Shot 学习设置中，模型会被给予几个示例，以帮助模型理解任务，并生成正确的响应。<br />在One-Shot学习设置中，模型会被给予一个示例，以帮助模型理解任务，并生成正确的响应。<br />在 Zero-Shot 学习设置中，模型只根据任务的描述生成响应，不需要任何示例。
```python
# 1. 创建一些示例
samples = [
  {
    "flower_type": "玫瑰",
    "occasion": "爱情",
    "ad_copy": "玫瑰，浪漫的象征，是你向心爱的人表达爱意的最佳选择。"
  },
  {
    "flower_type": "康乃馨",
    "occasion": "母亲节",
    "ad_copy": "康乃馨代表着母爱的纯洁与伟大，是母亲节赠送给母亲的完美礼物。"
  },
  {
    "flower_type": "百合",
    "occasion": "庆祝",
    "ad_copy": "百合象征着纯洁与高雅，是你庆祝特殊时刻的理想选择。"
  },
  {
    "flower_type": "向日葵",
    "occasion": "鼓励",
    "ad_copy": "向日葵象征着坚韧和乐观，是你鼓励亲朋好友的最好方式。"
  }
]

# 2. 创建一个提示模板
from langchain.prompts.prompt import PromptTemplate
template="鲜花类型: {flower_type}\n场合: {occasion}\n文案: {ad_copy}"
prompt_sample = PromptTemplate(input_variables=["flower_type", "occasion", "ad_copy"], 
                               template=template)
print(prompt_sample.format(**samples[0]))

# 3. 创建一个FewShotPromptTemplate对象
from langchain.prompts.few_shot import FewShotPromptTemplate
prompt = FewShotPromptTemplate(
    examples=samples,
    example_prompt=prompt_sample,
    suffix="鲜花类型: {flower_type}\n场合: {occasion}",
    input_variables=["flower_type", "occasion"]
)
print(prompt.format(flower_type="野玫瑰", occasion="爱情"))

# 4. 把提示传递给大模型
import os
os.environ["OPENAI_API_KEY"] = '你的Open AI Key'
from langchain.llms import OpenAI
model = OpenAI(model_name='text-davinci-003')
result = model(prompt.format(flower_type="野玫瑰", occasion="爱情"))
print(result)
```
如果你给出的示例过多，可使用`示例选择器`来进行筛选。
<a name="E98PP"></a>
## Chain Of Thought (COT) 思维链
> 如果生成一系列的中间推理步骤，就能够显著提高大型语言模型进行复杂推理的能力。

一个Few-Shot的COT示例，在示例中给出了思考过程<br />
![20241220155243](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20241220155243.png)
![20241220155314](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20241220155314.png)

```python
# 设置环境变量和API密钥
import os
os.environ["OPENAI_API_KEY"] = '你的OpenAI API Key'

# 创建聊天模型
from langchain.chat_models import ChatOpenAI
llm = ChatOpenAI(temperature=0)

# 设定 AI 的角色和目标
role_template = "你是一个为花店电商公司工作的AI助手, 你的目标是帮助客户根据他们的喜好做出明智的决定"

# CoT 的关键部分，AI 解释推理过程，并加入一些先前的对话示例（Few-Shot Learning）
cot_template = """
作为一个为花店电商公司工作的AI助手，我的目标是帮助客户根据他们的喜好做出明智的决定。 

我会按部就班的思考，先理解客户的需求，然后考虑各种鲜花的涵义，最后根据这个需求，给出我的推荐。
同时，我也会向客户解释我这样推荐的原因。

示例 1:
  人类：我想找一种象征爱情的花。
  AI：首先，我理解你正在寻找一种可以象征爱情的花。在许多文化中，红玫瑰被视为爱情的象征，这是因为它们的红色通常与热情和浓烈的感情联系在一起。因此，考虑到这一点，我会推荐红玫瑰。红玫瑰不仅能够象征爱情，同时也可以传达出强烈的感情，这是你在寻找的。

示例 2:
  人类：我想要一些独特和奇特的花。
  AI：从你的需求中，我理解你想要的是独一无二和引人注目的花朵。兰花是一种非常独特并且颜色鲜艳的花，它们在世界上的许多地方都被视为奢侈品和美的象征。因此，我建议你考虑兰花。选择兰花可以满足你对独特和奇特的要求，而且，兰花的美丽和它们所代表的力量和奢侈也可能会吸引你。
"""
from langchain.prompts import ChatPromptTemplate, HumanMessagePromptTemplate, SystemMessagePromptTemplate
system_prompt_role = SystemMessagePromptTemplate.from_template(role_template)
system_prompt_cot = SystemMessagePromptTemplate.from_template(cot_template)

# 用户的询问
human_template = "{human_input}"
human_prompt = HumanMessagePromptTemplate.from_template(human_template)

# 将以上所有信息结合为一个聊天提示
chat_prompt = ChatPromptTemplate.from_messages([system_prompt_role, system_prompt_cot, human_prompt])

prompt = chat_prompt.format_prompt(human_input="我想为我的女朋友购买一些花。她喜欢粉色和紫色。你有什么建议吗?").to_messages()

# 接收用户的询问，返回回答结果
response = llm(prompt)
print(response)
```
<a name="VEMWX"></a>
## Tree Of Thought(TOT) 思维树
ToT 是一种解决复杂问题的框架，它在需要多步骤推理的任务中，引导语言模型搜索一棵由连贯的语言序列（解决问题的中间步骤）组成的思维树，而不是简单地生成一个答案。ToT 框架的核心思想是：让模型生成和评估其思维的能力，并将其与搜索算法（如广度优先搜索和深度优先搜索）结合起来，进行系统性地探索和验证。<br />
![20241220155341](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20241220155341.png)
ToT 框架为每个任务定义具体的思维步骤和每个步骤的候选项数量。例如，要解决一个数学推理任务，先把它分解为 3 个思维步骤，并为每个步骤提出多个方案，并保留最优的 5 个候选方案。然后在多条思维路径中搜寻最优的解决方案。<br />这种方法的优势在于，模型可以通过观察和评估其自身的思维过程，更好地解决问题，而不仅仅是基于输入生成输出。这对于需要深度推理的复杂任务非常有用。此外，通过引入强化学习、集束搜索等技术，可以进一步提高搜索策略的性能，并让模型在解决新问题或面临未知情况时有更好的表现。<br />TOT的参考仓库：[https://github.com/kyegomez/tree-of-thoughts](https://github.com/kyegomez/tree-of-thoughts)

