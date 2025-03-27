# tool call

## function call of openai
- “函数调用”指的是 LLM 从用户提示中推断出需要执行的正确函数，以及传递给该函数的正确参数的能力
> 函数调用允许 LLM 根据用户输入，生成调用特定函数的指令及其参数，以结构化的 JSON 格式返回。​这使得模型能够与外部 API、数据库等系统交互，获取实时数据或执行特定操作。
- 定义函数
- 定义函数对应的json标准
```
{
          "type": "function",
          "function": {
              "name": "get_current_weather",
              "description": "获取指定位置的当前天气",
              "parameters": {
                  "type": "object",
                  "properties": {
                      "location": {
                          "type": "string",
                          "description": "城市名称，例如，北京",
                      },
                      "unit": {
                          "type": "string",
                          "enum": ["celsius", "fahrenheit"],
                          "description": "温度单位，摄氏度或华氏度",
                      },
                  },
                  "required": ["location"],
              },
          },
      }
```
- 将请求和tools定义传入open ai接口
- 模型更加该tools描述，返回调用信息
```
{
  "content": null,
  "role": "assistant",
  "function_call": null,
  "tool_calls": [
    {
      "id": "...",
      "function": {
        "arguments": "{\"location\":\"London\",\"unit\":\"celsius\"}",
        "name": "get_current_weather"
      },
      "type": "function"
    }
  ]
}
```
- 客户端根据该结构体调用函数
- 特点：模型需要支持tool调用,客户端定义函数和解析函数的调用

## langchain
> Langchain 与 OpenAI 的工具调用能力实现了无缝集成。Langchain 提供了 bind_tools() 方法，可以将 Langchain 的工具绑定到聊天模型上，从而在每次调用模型 API 时都包含工具的模式  1 。当 LLM 的响应中包含工具调用时，这些调用会作为 ToolCall 对象列表附加到相应的 AIMessage 或 AIMessageChunk 对象的 .tool_calls 属性中  1 。Langchain 能够处理不同 LLM 提供商在工具模式和工具调用方面的不同约定，为开发者提供了一个统一的接口  1 。此外，Langchain 还支持使用 Pydantic 来定义更复杂的工具输入  1 。开发者甚至可以强制模型调用特定的工具  2 。通过这些集成功能，Langchain 极大地简化了使用 OpenAI 工具调用的过程，使得开发者可以更加专注于定义工具的功能和构建智能代理的行为
>
## MCP
> MCP 基于三个核心原则：开放标准、跨平台兼容性和卓越的开发者体验（通过直观的 SDK） 1 。其架构采用客户端-服务器模型，涉及三个主要组件：主机（LLM 应用程序，例如 Claude Desktop）、客户端（主机应用程序内的连接器）和服务器（提供上下文和功能的外部服务） 2 。在服务器端，MCP 引入了三个主要的基元：工具（可执行的函数，允许 LLM 执行超出文本推理的操作）、资源（结构化数据，例如文件、日志或 API 响应，LLM 可以参考这些数据来生成输出）和提示（预定义的模板，用于规范交互，确保一致性和效率） 3 。在客户端，MCP 定义了根（Roots，控制服务器可以访问的主机文件系统或环境中的特定位置）和采样（Sampling，一种允许 MCP 服务器向客户端请求 LLM 完成的功能，反转了传统的客户端-服务器关系） 3

> 函数/工具调用主要侧重于使 LLM 能够通过函数/API 调用触发特定操作或检索数据 。相比之下，MCP 是一个更广泛的协议，旨在实现上下文和工具的标准化集成 。函数/工具调用通常是特定 LLM 提供商 API 的一项功能，而 MCP 作为一个开放标准，旨在实现跨平台兼容性 。
