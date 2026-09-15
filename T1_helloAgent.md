这次从零搭建 Agent，其实已经走完了一条非常典型、也很有学习价值的路径。可以把整个过程总结成“环境 → API → 工具 → Agent 循环 → 可靠性优化 → 交互体验”六个阶段。

1. **先搭建基础开发环境。** 你最后选择的是 Windows + CMD/PowerShell + VS Code + Python 虚拟环境。核心结构是：项目目录 `hello-agents`，项目内创建 `venv`，再用 `venv\Scripts\activate.bat` 激活。经验是：项目目录和虚拟环境要分清，最好保持 `项目根目录/venv` 的结构，不要把整个项目文件夹本身当成虚拟环境。Python 多版本并存时，要用 `where python` 或 `py -0p` 确认当前到底调用的是哪个解释器。

2. **用 `.env` 管理 API Key，而不是写死在代码里。** 你配置了 `OPENAI_API_KEY`、`OPENAI_BASE_URL`、`MODEL_NAME` 和 `TAVILY_API_KEY`，再用 `python-dotenv` 的 `load_dotenv()` 读取。这个步骤里最典型的问题是：`.env` 没保存、变量名不一致、API Key 后面混入中文说明文字，都会导致读取失败或请求报错。一个很重要的经验是：先单独写 `test_env.py` 验证变量是否读取成功，再进入 Agent 主程序，不要把所有问题混在一起排查。

3. **先单独测试每个外部能力，再组合成 Agent。** 你分别测试了 AIHubMix 的 LLM 调用、Tavily 搜索、天气 API。这个顺序非常重要。因为如果直接跑 Agent，一旦报错，很难判断是 LLM、搜索工具、天气接口还是 Agent 逻辑有问题。以后做任何 Agent 都推荐遵循这个原则：**每个 Tool 先单测，再集成。**

4. **LLM 接口要优先验证最小可用调用。** 你通过 OpenAI-compatible SDK 成功调用 AIHubMix，核心配置是 `base_url="https://aihubmix.com/v1"`、正确的模型名和 Key。过程中遇到的 `multipart/form-data` 报错，本质是配置写错，而不是 Agent 本身的问题。经验是：先跑一个最简单的 `test_llm.py`，确认“输入一句话 → 返回一句话”成功，再把同样的 client 配置搬进 Agent。

5. **手写 ReAct Agent 的核心，是循环：思考 → 行动 → 工具结果 → 再思考。** 你的第一版实际上是一个手工实现的 ReAct Agent：System Prompt 规定 `Thought` 和 `Action` 格式，模型输出类似 `get_weather(city="北京")`，程序用正则解析，再执行对应 Python 函数，把结果作为 `Observation` 放回历史中，直到模型输出 `Finish[...]`。这非常适合学习 Agent 的本质，因为你能清楚看到：模型不是“直接回答”，而是在“选择工具—读取结果—继续推理”。

6. **最大的 Agent 可靠性问题，是模型会跳过工具、自己猜实时事实。** 你第一次运行时，模型直接说“北京今天是晴”，然后调用景点工具，却没有调用 `get_weather`。这是非常典型的 Agent 问题：LLM 会“看起来合理地作弊”。真正的经验是：**Prompt 约束不能代替程序约束。** 对实时天气、价格、新闻、数据库结果等，应该强制规定必须调用工具，并在代码层记录状态，比如“只有天气工具成功后才能调用景点工具”。

7. **外部 Tool 自身可能不稳定，不能把 Tool 失败误判为 Agent 失败。** 你用 `wttr.in` 时遇到了超时、500 和 `location not found`。这并不是 Python 或 Agent 错误，而是外部服务异常。后来换成 Open-Meteo 后，天气工具正常工作。这里的经验非常重要：Agent 的工具层要有超时、HTTP 错误、解析错误等异常处理，并尽量返回结构化错误，例如 `ERROR: weather service unavailable`，而不是直接让整个程序崩掉。

8. **工具返回值最好尽量结构化，而不是模糊自然语言。** 你后来天气工具返回类似：`WEATHER_OK: 城市=北京; 天气=晴; 气温=22.3°C; ...`，这比单纯“北京天气不错”好得多。结构越清晰，Agent 越不容易误解。进一步可以直接返回字典或 JSON，而不是字符串，这也是未来从“教学版 Agent”向工程化 Agent 过渡的一步。

9. **搜索类工具的 Prompt/query 也会影响结果。** 你发现 Tavily 对 `weather="晴"` 和 `weather="晴天"` 返回结果不同。这说明 Tool 并不是一个固定数据库查询，而是搜索模型/搜索 API。经验是：工具输入本身也需要设计，尽量用完整、自然、明确的查询，例如“北京今天晴天，推荐 3 个适合这种天气游览的景点，并说明理由”，比简单拼接关键词更稳定。

10. **当前这种 `Thought/Action + regex` 是很好的学习方式，但不是最终工程形态。** 你现在依赖正则解析：`Action: get_weather(...)`。这种方案很容易被单双引号、换行、Markdown、格式偏差破坏。长期更推荐原生 Tool Calling / Function Calling，让模型返回结构化工具名和参数，而不是自己从文本里解析。换句话说，这一版的价值是“理解 Agent 原理”，下一阶段要学的是“结构化 Tool Calling”。

11. **交互体验也很重要。** 一开始你的用户问题是直接写死在 Python 文件里的，所以更像脚本，不像聊天 Agent。下一步把 `user_prompt = "..."` 改成 `input()`，再进一步加 `while True`，就能实现 CMD 中的连续对话。再往后增加 conversation history，才能真正理解“那上海呢？”、“换一个景点”等上下文式对话。

12. **调试日志和用户界面最好分开。** 学习阶段显示 `Thought / Action / Observation` 很有帮助，但真正用户使用时，只应该看到“你：”“Agent：”。可以加一个 `DEBUG = True/False` 开关。调试模式看 Agent 内部流程，正常模式只展示最终结果。

如果把这次经验压缩成一套以后可以复用的“从零搭 Agent 方法”，就是：

```text
确定任务
→ 拆成 LLM + Tools
→ 建 Python 虚拟环境
→ 用 .env 管理密钥
→ 单测 LLM
→ 单测每个 Tool
→ 写 Agent System Prompt
→ 建立 Agent Loop
→ 加 Tool 异常处理
→ 加程序级工具调用约束
→ 加对话输入
→ 再升级到 Structured Tool Calling
```

这次最值得记住的三个经验是：

- **不要一上来就写完整 Agent，先单测每一个组件。**

- **不要相信模型“应该会调用工具”，要在代码层约束它。**

- **Agent 的问题不只来自 LLM，更多时候来自 Tool、API、状态管理和格式解析。**

你现在已经不只是“把教程跑通”，而是实际经历了一次小型 Agent 工程调试流程。下一步如果继续学，最有价值的是把这个旅行 Agent 改造成 **真正的 Tool Calling 版本**，这样你会非常清楚地看到“手写 ReAct”和“工程化 Agent”之间的区别。

![](C:/Users/HUAWEI/AppData/Roaming/marktext/images/2026-09-15-23-45-50-image.png)
