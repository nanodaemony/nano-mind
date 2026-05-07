
[https://mp.weixin.qq.com/s/UfzmzJ3zTOQ45ppRy5D6kA](https://mp.weixin.qq.com/s/UfzmzJ3zTOQ45ppRy5D6kA)

## 一、前言

说实话，当我看到使用 MCP 服务还需要手动复制粘贴 JSON 的时候，包括现在很多 MCP 服务只有工具，没有资源和提示时，我认为 MCP 还不太成熟。

随着今年 “智能体” 的大爆发，使用**工具**的诉求越来越强烈。随着 MCP 服务市场、MCP 自动配置功能的出现，MCP 的使用门槛正在降低，越来越多的服务宣布支持 MCP 协议，开始要爆发的趋势。

本文将先尽量用相对通俗的语言和图形**介绍基本的理论**，然后以 ModelScope 的 MCP 市场，Cherry Studio 客户端为例，为大家展示 MCP 的过程，让大家更加有体感，更好地理解和掌握 MCP，最后分享一点资料和观点。

## 二、理论基础

**RAG 是为了让大模型获取足够的上下文，Function Calling 是为了让模型使用工具**，他们和 MCP 有或多或少的联系，因此接下来先介绍前面两个概念再介绍 MCP。

### RAG

**RAG（Retrieval Augmented Generation ，检索增强生成）**，我们**不需要训练和微调大模型，只需要提供和用户提问相关的额外的信息到提示词中，从而可以获得更高质量的回答**。

![[yuque-1778165507586-1.png]]

[图片来源](about:blank)

通常，需要**将资料通过嵌入模型生成服务转化为向量，然后存储到向量数据库中**。**当用户提问时，将用户的问题向量化从向量数据库中进行相似度匹配出 TOP N 个片段，拼接成新的提示词发送给大模型，大模型就可以结合你的资料更好地回答问题了**。

当然， RAG 还有很多变种，如 `Agentic RAG` 、`Modular RAG`、`Graph RAG` 等多种高级形式，这里就不展了。

### Function Calling

**Function Calling (函数调用) 是一种允许大型语言模型(LLM)根据用户输入识别它需要的工具并决定何时调用该工具的机制**。

![[yuque-1778165507617-2.gif]]

图片来源：ailydoseofds

基本工作原理如下：**LLM 接收用户的提示词，LLM 决定它需要的工具，执行方法调用，后端服务执行实际的请求给出处理结果，大语言模型根据处理结果生成最终给用户的回答**。

**不同的 API 需要封装成不同的方法，通常需要编写代码，很难在不同的平台灵活复用。**

参见百炼函数调用文档：`https://help.aliyun.com/zh/model-studio/tool-calling-overview/`

### 看看 MCP

**模型上下文协议（Model Context Protocol，简称 MCP）**是一个由 Anthropic 在 2024 年 11 月 25 日开源的新标准。**MCP 是一个开放标准，旨在连接 AI 助手与数据所在的系统，包括内容存储库、业务工具和开发环境。其目标是帮助前沿模型产生更好、更相关的响应**。

![[yuque-1778165507640-3.png]]

MCP 可以看作是 **AI 应用程序的 "USB-C端口"**。就像 USB-C 为连接设备与各种外设提供了标准化方式，**MCP为 AI 模型连接不同数据源和工具提供了标准化方法**。

![[yuque-1778165507668-4.gif]]

#### **MCP 的核心组件**

从技术角度看，MCP 遵循客户端-服务器架构，其中**主机应用可以连接到多个服务器**。

MCP 有三个关键组件：

- **主机(Host)**
- **客户端(Client)**
- **服务器(Server)**

具体功能如下：

- **主机：代表任何****提供 AI 交互环境的应用程序****(如 Claude 桌面版、Cursor)，它能访问工具和数据，并运行 MCP 客户端。**
- **MCP 客户端：****在主机内运行，使其能与 MCP 服务器通信****。**
- **MCP 服务器：****暴露特定功能并提供数据访问，例如：**

- **工具**：使 LLM 能通过服务器执行操作。
- **资源**：向 LLM 公开服务器中的数据和内容。
- **提示**：创建可重用的提示模板和工作流。

#### **MCP 相比传统 API 的优势**

传统 API ：

- 如果 API **最初需要两个参数**(例如天气服务的位置和日期)，用户集成应用程序会发送带有这些确切参数的请求。

![[yuque-1778165507739-5.png]]

- 如果后来添加第三个必需参数(例如温度单位，如摄氏度或华氏度)，API的合约就会改变。这意味着**所有 API 用户必须更新代码以包含新参数。如果不更新，请求可能失败、返回错误或提供不完整结果**。

![[yuque-1778165507752-6.png]]

MCP 的设计解决了这个问题：

- MCP 引入了与传统 API 截然不同的动态灵活方法。
- 当客户端(如 Claude 桌面版)连接到 MCP 服务器(如您的天气服务)时，它发送初始请求以**了解服务器的能力**。

![[yuque-1778165507766-7.png]]

- **服务器响应其可用工具、资源、提示和参数的详细信息**。例如，如果您的天气 API 最初支持位置和日期，服务器会将这些作为其功能的一部分进行通信。

![[yuque-1778165507777-8.png]]

- **如果您稍后添加单位参数，MCP 服务器可以在下一次交换期间动态更新其功能描述**。客户端不需要硬编码或预定义参数—只需查询服务器的当前功能并相应地适应。
- 这样，**客户端可以及时调整其行为，使用更新的功能(例如在请求中包含单位)，无需重写或重新部署代码**。

![[yuque-1778165507790-9.png]]

其实，MCP 也是用了 “**中间层**” 的思想，**让大模型使用工具标准化，让大模型调用工具更方便，焕发出新的生机**。

#### **MCP 的过程**

![[yuque-1778165507807-10.gif]]

首先需要**在主机上自动或手动配置 MCP 服务**，当用户输入问题时， **MCP 客户端让大语言模型选择 MCP 工具，大模型选择好 MCP 工具以后， MCP 客户端寻求用户同意（很多产品支持配置自动同意），MCP 客户端请求 MCP 服务器， MCP 服务调用工具并将工具的结果返回给 MCP 客户端， MCP 客户端将模型调用结果和用户的查询发送给大语言模型，大语言模型组织答案给用户**。

**其实 RAG 、Function Call 和 MCP 本质上都是一样，都是为了借助外部工具帮助大模型完成更复杂的事情。**

## 三、“看见” MCP 的过程

### ModelScope

![[yuque-1778165507825-11.png]]

我们来到魔搭的 MCP 市场：`https://modelscope.cn/mcp`，这里只是为了演示，其他的 MCP 广场也是一样的。

![[yuque-1778165507846-12.png]]

登录以后，选择一个 MCP 服务，点击右侧的 “连接” 按钮，就可以获取 MCP 服务的 `SSE URL`。

### Cherry Studio

`Cherry Studio` 官网：`https://cherry-ai.com/`

![[yuque-1778165507888-13.png]]

在右上角点击开启，**就可以看到该 MCP 服务提供的工具、提示和资源**。

![[yuque-1778165507906-14.png]]

可以**单独开启和关闭每个可用的工具**。

![[yuque-1778165507935-15.png]]

每个 MCP 服务手动配置太麻烦，大家也可以点击 “同步服务器”，配置 ModelScope 的 API 令牌，配置之后就可以**一键同步 ModelScope 上开启的 MCP 服务了**。

### 开发者模式

![[yuque-1778165507948-16.png]]

输入想要问的问题，开启需要的 MCP 服务。

为了看见 MCP 的执行过程，还需要通过 `Ctrl+Shift+I`（Mac端：`Command+Option+I`）打开**开发者模式**。

![[yuque-1778165507967-17.png]]

**第一次请求，发送系统提示词，包括大模型可以选择的工具和要求等，然后给出用户的问题**。

![[yuque-1778165507993-18.png]]

可以看到第二次调用，**模型选择使用 infoq 工具获得资讯后组织语言展示给用户**。

`index = 0` 的系统提示词的内容：

```
{
  "role": "system",
  "content": "In this environment you have access to a set of tools you can use to answer the user's question. You can use one tool per message, and will receive the result of that tool use in the user's response. You use tools step-by-step to accomplish a given task, with each tool use informed by the result of the previous tool use.\n\n## Tool Use Formatting\n\nTool use is formatted using XML-style tags. The tool name is enclosed in opening and closing tags, and each parameter is similarly enclosed within its own set of tags. Here's the structure:\n\n<tool_use>\n  <name>{tool_name}</name>\n  <arguments>{json_arguments}</arguments>\n</tool_use>\n\nThe tool name should be the exact name of the tool you are using, and the arguments should be a JSON object containing the parameters required by that tool. For example:\n<tool_use>\n  <name>python_interpreter</name>\n  <arguments>{\"code\": \"5 + 3 + 1294.678\"}</arguments>\n</tool_use>\n\nThe user will respond with the result of the tool use, which should be formatted as follows:\n\n<tool_use_result>\n  <name>{tool_name}</name>\n  <result>{result}</result>\n</tool_use_result>\n\nThe result should be a string, which can represent a file or any other output type. You can use this result as input for the next action.\nFor example, if the result of the tool use is an image file, you can use it in the next action like this:\n\n<tool_use>\n  <name>image_transformer</name>\n  <arguments>{\"image\": \"image_1.jpg\"}</arguments>\n</tool_use>\n\nAlways adhere to this format for the tool use to ensure proper parsing and execution.\n\n## Tool Use Examples\n\nHere are a few examples using notional tools:\n---\nUser: Generate an image of the oldest person in this document.\n\nAssistant: I can use the document_qa tool to find out who the oldest person is in the document.\n<tool_use>\n  <name>document_qa</name>\n  <arguments>{\"document\": \"document.pdf\", \"question\": \"Who is the oldest person mentioned?\"}</arguments>\n</tool_use>\n\nUser: <tool_use_result>\n  <name>document_qa</name>\n  <result>John Doe, a 55 year old lumberjack living in Newfoundland.</result>\n</tool_use_result>\n\nAssistant: I can use the image_generator tool to create a portrait of John Doe.\n<tool_use>\n  <name>image_generator</name>\n  <arguments>{\"prompt\": \"A portrait of John Doe, a 55-year-old man living in Canada.\"}</arguments>\n</tool_use>\n\nUser: <tool_use_result>\n  <name>image_generator</name>\n  <result>image.png</result>\n</tool_use_result>\n\nAssistant: the image is generated as image.png\n\n---\nUser: \"What is the result of the following operation: 5 + 3 + 1294.678?\"\n\nAssistant: I can use the python_interpreter tool to calculate the result of the operation.\n<tool_use>\n  <name>python_interpreter</name>\n  <arguments>{\"code\": \"5 + 3 + 1294.678\"}</arguments>\n</tool_use>\n\nUser: <tool_use_result>\n  <name>python_interpreter</name>\n  <result>1302.678</result>\n</tool_use_result>\n\nAssistant: The result of the operation is 1302.678.\n\n---\nUser: \"Which city has the highest population , Guangzhou or Shanghai?\"\n\nAssistant: I can use the search tool to find the population of Guangzhou.\n<tool_use>\n  <name>search</name>\n  <arguments>{\"query\": \"Population Guangzhou\"}</arguments>\n</tool_use>\n\nUser: <tool_use_result>\n  <name>search</name>\n  <result>Guangzhou has a population of 15 million inhabitants as of 2021.</result>\n</tool_use_result>\n\nAssistant: I can use the search tool to find the population of Shanghai.\n<tool_use>\n  <name>search</name>\n  <arguments>{\"query\": \"Population Shanghai\"}</arguments>\n</tool_use>\n\nUser: <tool_use_result>\n  <name>search</name>\n  <result>26 million (2019)</result>\n</tool_use_result>\nAssistant: The population of Shanghai is 26 million, while Guangzhou has a population of 15 million. Therefore, Shanghai has the highest population.\n\n\n## Tool Use Available Tools\nAbove example were using notional tools that might not exist for you. You only have access to these tools:\n<tools>\n\n<tool>\n  <name>f_u-LsDUyQWsNFAUjL-_Vi</name>\n  <description>获取 36 氪热榜，提供创业、商业、科技领域的热门资讯，包含投融资动态、新兴产业分析和商业模式创新信息</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"type\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"hot\",\"description\":\"人气榜\"},{\"type\":\"string\",\"const\":\"video\",\"description\":\"视频榜\"},{\"type\":\"string\",\"const\":\"comment\",\"description\":\"热议榜\"},{\"type\":\"string\",\"const\":\"collect\",\"description\":\"收藏榜\"}],\"default\":\"hot\",\"description\":\"分类\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fyQ3DOUpLJDGPLCjslT6P1</name>\n  <description>获取 9to5Mac 苹果相关新闻，包含苹果产品发布、iOS 更新、Mac 硬件、应用推荐及苹果公司动态的英文资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fBOukVLVbqIh69earzcYca</name>\n  <description>获取 BBC 新闻，提供全球新闻、英国新闻、商业、政治、健康、教育、科技、娱乐等资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"category\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"\",\"description\":\"热门新闻\"},{\"type\":\"string\",\"const\":\"world\",\"description\":\"国际\"},{\"type\":\"string\",\"const\":\"uk\",\"description\":\"英国\"},{\"type\":\"string\",\"const\":\"business\",\"description\":\"商业\"},{\"type\":\"string\",\"const\":\"politics\",\"description\":\"政治\"},{\"type\":\"string\",\"const\":\"health\",\"description\":\"健康\"},{\"type\":\"string\",\"const\":\"education\",\"description\":\"教育\"},{\"type\":\"string\",\"const\":\"science_and_environment\",\"description\":\"科学与环境\"},{\"type\":\"string\",\"const\":\"technology\",\"description\":\"科技\"},{\"type\":\"string\",\"const\":\"entertainment_and_arts\",\"description\":\"娱乐与艺术\"}],\"default\":\"\"},\"edition\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"\"},{\"type\":\"string\",\"const\":\"uk\",\"description\":\"UK\"},{\"type\":\"string\",\"const\":\"us\",\"description\":\"US & Canada\"},{\"type\":\"string\",\"const\":\"int\",\"description\":\"Rest of the world\"}],\"default\":\"\",\"description\":\"版本，仅对 `category` 为空有效\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fTU2iCLtmsEWljy_HbEhCa</name>\n  <description>获取哔哩哔哩视频排行榜，包含全站、动画、音乐、游戏等多个分区的热门视频，反映当下年轻人的内容消费趋势</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"type\":{\"anyOf\":[{\"type\":\"number\",\"const\":0,\"description\":\"全站\"},{\"type\":\"number\",\"const\":1,\"description\":\"动画\"},{\"type\":\"number\",\"const\":3,\"description\":\"音乐\"},{\"type\":\"number\",\"const\":4,\"description\":\"游戏\"},{\"type\":\"number\",\"const\":5,\"description\":\"娱乐\"},{\"type\":\"number\",\"const\":188,\"description\":\"科技\"},{\"type\":\"number\",\"const\":119,\"description\":\"鬼畜\"},{\"type\":\"number\",\"const\":129,\"description\":\"舞蹈\"},{\"type\":\"number\",\"const\":155,\"description\":\"时尚\"},{\"type\":\"number\",\"const\":160,\"description\":\"生活\"},{\"type\":\"number\",\"const\":168,\"description\":\"国创相关\"},{\"type\":\"number\",\"const\":181,\"description\":\"影视\"}],\"default\":0,\"description\":\"排行榜分区\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>f_OULD1Qnu5wPblKHFTzYw</name>\n  <description>获取豆瓣实时热门榜单，提供当前热门的图书、电影、电视剧、综艺等作品信息，包含评分和热度数据</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"type\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"subject\",\"description\":\"图书、电影、电视剧、综艺等\"},{\"type\":\"string\",\"const\":\"movie\",\"description\":\"电影\"},{\"type\":\"string\",\"const\":\"tv\",\"description\":\"电视剧\"}],\"default\":\"subject\"},\"start\":{\"type\":\"integer\",\"default\":0},\"count\":{\"type\":\"integer\",\"default\":10}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fL5c-CwZ3bUckaatY_MC8Z</name>\n  <description>获取抖音热搜榜单，展示当下最热门的社会话题、娱乐事件、网络热点和流行趋势</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>flPxhdB8Wyw8DDuban0pMa</name>\n  <description>获取机核网游戏相关资讯，包含电子游戏评测、玩家文化、游戏开发和游戏周边产品的深度内容</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fdj1bDLATGns6ZdclA68UF</name>\n  <description>获取爱范儿科技快讯，包含最新的科技产品、数码设备、互联网动态等前沿科技资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"limit\":{\"type\":\"integer\",\"default\":20},\"offset\":{\"type\":\"integer\",\"default\":0}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>f0Uk75Pbfsx0rfdD725CYI</name>\n  <description>获取 InfoQ 技术资讯，包含软件开发、架构设计、云计算、AI等企业级技术内容和前沿开发者动态</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"region\":{\"type\":\"string\",\"enum\":[\"cn\",\"global\"],\"default\":\"cn\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fu70DSpIpWTVlq11tEKJZd</name>\n  <description>获取掘金文章榜，包含前端开发、后端技术、人工智能、移动开发及技术架构等领域的高质量中文技术文章和教程</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"category_id\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"6809637769959178254\",\"description\":\"后端\"},{\"type\":\"string\",\"const\":\"6809637767543259144\",\"description\":\"前端\"},{\"type\":\"string\",\"const\":\"6809635626879549454\",\"description\":\"Android\"},{\"type\":\"string\",\"const\":\"6809635626661445640\",\"description\":\"iOS\"},{\"type\":\"string\",\"const\":\"6809637773935378440\",\"description\":\"人工智能\"},{\"type\":\"string\",\"const\":\"6809637771511070734\",\"description\":\"开发工具\"},{\"type\":\"string\",\"const\":\"6809637776263217160\",\"description\":\"代码人生\"},{\"type\":\"string\",\"const\":\"6809637772874219534\",\"description\":\"阅读\"}],\"default\":\"6809637769959178254\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fZMjSXkog94GRA7VoZpJtw</name>\n  <description>获取网易新闻热点榜，包含时政要闻、社会事件、财经资讯、科技动态及娱乐体育的全方位中文新闻资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>f5s3U6zXATp3L6wpAwNcn4</name>\n  <description>获取纽约时报新闻，包含国际政治、经济金融、社会文化、科学技术及艺术评论的高质量英文或中文国际新闻资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"region\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"cn\",\"description\":\"中文\"},{\"type\":\"string\",\"const\":\"global\",\"description\":\"全球\"}],\"default\":\"cn\"},\"section\":{\"type\":\"string\",\"default\":\"HomePage\",\"description\":\"分类，当 `region` 为 `cn` 时无效。可选值: Africa, Americas, ArtandDesign, Arts, AsiaPacific, Automobiles, Baseball, Books/Review, Business, Climate, CollegeBasketball, CollegeFootball, Dance, Dealbook, DiningandWine, Economy, Education, EnergyEnvironment, Europe, FashionandStyle, Golf, Health, Hockey, HomePage, Jobs, Lens, MediaandAdvertising, MiddleEast, MostEmailed, MostShared, MostViewed, Movies, Music, NYRegion, Obituaries, PersonalTech, Politics, ProBasketball, ProFootball, RealEstate, Science, SmallBusiness, Soccer, Space, Sports, SundayBookReview, Sunday-Review, Technology, Television, Tennis, Theater, TMagazine, Travel, Upshot, US, Weddings, Well, World, YourMoney\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fSodMRu-GB5-mpBxBvxc2D</name>\n  <description>获取什么值得买热门，包含商品推荐、优惠信息、购物攻略、产品评测及消费经验分享的实用中文消费类资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"unit\":{\"anyOf\":[{\"type\":\"number\",\"const\":1,\"description\":\"今日热门\"},{\"type\":\"number\",\"const\":7,\"description\":\"周热门\"},{\"type\":\"number\",\"const\":30,\"description\":\"月热门\"}],\"default\":1}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fnY78eScFYQ-6xtUUQg8fB</name>\n  <description>获取少数派热榜，包含数码产品评测、软件应用推荐、生活方式指南及效率工作技巧的优质中文科技生活类内容</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"tag\":{\"type\":\"string\",\"enum\":[\"热门文章\",\"应用推荐\",\"生活方式\",\"效率技巧\",\"少数派播客\"],\"default\":\"热门文章\",\"description\":\"分类\"},\"limit\":{\"type\":\"integer\",\"default\":40}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fFlnyuBJfPLifqYRkW94Lh</name>\n  <description>获取腾讯新闻热点榜，包含国内外时事、社会热点、财经资讯、娱乐动态及体育赛事的综合性中文新闻资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"page_size\":{\"type\":\"integer\",\"default\":20}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>f0YfISsRrFbNGGwO3XVCFY</name>\n  <description>获取澎湃新闻热榜，包含时政要闻、财经动态、社会事件、文化教育及深度报道的高质量中文新闻资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fIKw7RiYt2oAptPSBtelf-</name>\n  <description>获取 The Verge 新闻，包含科技创新、数码产品评测、互联网趋势及科技公司动态的英文科技资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fH1Xj1bmY_qWxS6b8BoSCK</name>\n  <description>获取今日头条热榜，包含时政要闻、社会事件、国际新闻、科技发展及娱乐八卦等多领域的热门中文资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fIoNHXnzDif1o8sJSRpVBu</name>\n  <description>获取微博热搜榜，包含时事热点、社会现象、娱乐新闻、明星动态及网络热议话题的实时热门中文资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fX_nTem-9mBD-9Th_ZVRdA</name>\n  <description>获取微信读书排行榜，包含热门小说、畅销书籍、新书推荐及各类文学作品的阅读数据和排名信息</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"category\":{\"anyOf\":[{\"type\":\"string\",\"const\":\"rising\",\"description\":\"飙升榜\"},{\"type\":\"string\",\"const\":\"hot_search\",\"description\":\"热搜榜\"},{\"type\":\"string\",\"const\":\"newbook\",\"description\":\"新书榜\"},{\"type\":\"string\",\"const\":\"general_novel_rising\",\"description\":\"小说榜\"},{\"type\":\"string\",\"const\":\"all\",\"description\":\"总榜\"}],\"default\":\"rising\",\"description\":\"排行榜分区\"}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n\n<tool>\n  <name>fzIq8ORLIZHJLRU6pJt3O5</name>\n  <description>获取知乎热榜，包含时事热点、社会话题、科技动态、娱乐八卦等多领域的热门问答和讨论的中文资讯</description>\n  <arguments>\n    {\"type\":\"object\",\"properties\":{\"limit\":{\"type\":\"number\",\"default\":50}},\"additionalProperties\":false,\"$schema\":\"http://json-schema.org/draft-07/schema#\"}\n  </arguments>\n</tool>\n\n</tools>\n\n## Tool Use Rules\nHere are the rules you should always follow to solve your task:\n1. Always use the right arguments for the tools. Never use variable names as the action arguments, use the value instead.\n2. Call a tool only when needed: do not call the search agent if you do not need information, try to solve the task yourself.\n3. If no tool call is needed, just answer the question directly.\n4. Never re-do a tool call that you previously did with the exact same parameters.\n5. For tool use, MARK SURE use XML tag format as shown in the examples above. Do not use any other format.\n\n# User Instructions\n\n\nNow Begin! If you solve the task correctly, you will receive a reward of $1,000,000.\n"
}
```

主要包括：

- **工具使用格式**：使用 XML 风格的标签来调用工具，包括工具名称和 JSON 格式的参数。
- **工具使用示例**：展示了如何使用各种假设工具的例子，如文档问答、图像生成、Python 解释器等。
- **可用工具列表**：提供了多个实际可用的信息获取工具，主要是各种中英文新闻和内容平台的热门榜单。
- **工具使用规则**：强调了正确使用参数、仅在需要时调用工具、不重复调用等规则。
- 最后，提示词以一个**有趣的激励**结束，表示如果任务正确完成，将获得一百万美元奖励。

`index =` 的用户的内容：

```
{
  "role": "user",
  "content": "帮我搜索 模型上下文协议 MCP 相关的最新技术文章"
}
```

`index = 2` 的助手的内容：

```
{
  "role": "assistant",
  "content": "为了找到关于模型上下文协议（MCP）的最新技术文章，我将使用一些可用的工具来搜索相关资讯。首先，我可以从InfoQ开始，因为它经常发布企业级技术内容和前沿开发者动态。\n\n<tool_use>\n  <name>f0Uk75Pbfsx0rfdD725CYI</name>\n  <arguments>{\"region\":\"global\"}</arguments>\n</tool_use>"
}
```

`index = 3` 的 用户的内容：

```
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "Here is the result of tool call f0Uk75Pbfsx0rfdD725CYI:"
    },
    {
      "type": "text",
      "text": "[{\"type\":\"text\",\"text\":\"<title>Docker Desktop 4.40 Introduces Model Runner to Run LLMs Locally Expanding its AI Capabilities</title>\\n<description> 省略其他}]"
    }
  ]
}
```

![[yuque-1778165508034-19.png]]

**第三次调用主要是为了给话题起标题**。

大家可以多开启几个工具或者多问几次，结合理论部分的讲解，会对 MCP 的过程会有更深刻的体会。

## 四、MCP 市场和 MCP 学习经验

### MCP 市场

MCP 市场有很多，国内常见的有 ModelScope 和 **阿里云百炼**。

![[yuque-1778165508080-20.png]]

`ModelScope`：`https://modelscope.cn/mcp`

![[yuque-1778165508115-21.png]]

百炼 MCP 市场：`https://bailian.console.aliyun.com/?tab=mcp#/mcp-market`

### MCP 学习技巧

![[yuque-1778165508132-22.png]]

大家还可以使用 `Qwen` （`https://chat.qwen.ai`）深入研究功能来系统学习 MCP。

![[yuque-1778165508144-23.png]]

感兴趣可以 Clone Cherry Studio 的源码，使用通义灵码、 Cursor 等 AI 编程工具，打开仓库：`https://github.com/CherryHQ/cherry-studio` 直接对你关心的问题进行提问，学习效果更好。

![[yuque-1778165508163-24.png]]

## 五、观点

### 一些问题

**问题 1：MCP 服务的配置、开关通常需要手动操作，使用方式还不够智能。这有点和 ChatGPT 的插件功能很相似，用户需要手动选择想要开启的插件，AI 还不能自动选择。**在大模型上下文和能力没有完全解决之前， MCP 下一步应该需要支持场景化（剧本式）开启和关闭，而不是全局的手动开启和关闭，开启太多 Tokens 消耗大，开启太少可能影响功能的使用，某个场景在特定环节能够用到的 MCP 是相对确定的，需要类似工作流编排的形式来解决。

**问题 2：如果开启大量的 MCP 服务，客户端如果第一次将所有工具信息都发给大模型让大模型来抉择，会浪费大量 Tokens。未来这部分可能需要用本地模型来实现。**

**问题 3：MCP 只是解决了协议的问题，工具的稳定性很重要，调用工具时服务不可用非常影响用户体验。应该有个工具可用性检测机制，不可用时及时下线。**

**问题 4：现在 MCP 服务的封装主流还是前端框架和 Python，Java 来封装 MCP 似乎不太方便或者说上手门槛略高，未来可能需要再增加一个中间层来吃掉这个复杂度。**

**问题 5：现在很多模型使用 MCP 的能力还不太强，有些场景哪怕适合调用某个 MCP 服务，有些模型也只会在需要明确告知使用 MCP 才会选择使用 MCP。**

### 关于 MCP 只是一种 “过渡” 的说法

很多人认为 MCP 只是一个过渡，因为**目前 MCP 设计存在一些局限性**，而且尽管 MCP 获得了 OpenAI、Google 等大厂的支持，但其成为**永久行业标准仍存在不确定性**。

我认为，MCP 确实存在一些缺点，但 MCP 确实为大模型更好地调用工具提供了一个非常好的协议。大模型想要做更复杂发挥出更大价值必然要学会使用工具，模型上下文协议非常有必要，只是未必一定是 Anthropic 这个版本。

MCP 可能是一个过渡，但这不是问题。其实 RAG 也存在诸多问题，可能也是一个过渡。其实，从更大尺度来看，我们以前用的电话、功能机、现在用的这些传统软件都是一种过渡。人，总是有时代局限性，比如我们手动挡的汽车和现在的自动驾驶没法比，也是一种过渡，但不妨碍它能够真正帮助到我们。

### 展望

当前，大模型现在还很难通过 MCP 实现人类一样的操作。比如 PPT 制作、视频剪辑，大模型还不能通过 MCP 实现类似真正和人类一样的智能的 “RPA” 操作。

随着越来越多的服务支持 MCP，随着模型的服务不断增强，面向 MCP 的业务流程编排会或许成为主流。随着 AI 使用工具的能力不断增强，未来，能够带来的帮助也会更大。

![[yuque-1778165508182-25.png]]

MCP 试图在解决模型使用工具的问题（就像人可以使用扳手、可以开车、可以使用的电脑等）。但是，想要完成非常复杂的事情，智能体之间必然需要协同（就像上下级之间的管理和执行，同事之间分工合作），智能体之间的标准协议非常重要。谷歌最近发布的 A2A 协议则试图解决智能体之间通信的协议问题。正如上图所示，两者可以结合使用，MCP 用户服务的注册和发现， A2A 用户智能体之间的通信。

AI 技术的发展比想象地要更快，正如 ChatGPT 刚出来时，很多人说它永远不会取代我们，它的知识不会更新，现在 AI 搜索、RAG 已经成为常态。

现在，模型虽然存在速度慢的问题，存在上下文长度限制问题，存在幻觉的问题，存在好用模型价格高的问题等，这些都将逐渐得到解决。 AI 将会更好地使用工具，Agent 也将更加智能， Agent 之间的协同也将更加通畅。
