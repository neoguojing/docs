# runnables

![runnable](./classes_runnables.png)


## 常用接口
- Invoked: 一个输入和输出
- Batched: 批处理接口
- Streamed: 流式接口
- Inspected: 获取输入输出类型的接口，get_input_schema，get_output_chema config_schema
- Composed: 复杂执行流程组合
- astream_events： 支持打印中间结果和最终结果

## 异步接口
ainvoke, abatch, astream, abatch_as_completed

## RunnableConfig
以上接口均有第二个参数config
- run_name： 名称
- run_id：	唯一id
- tags	
- metadata
- callbacks	回调
- max_concurrency	batch接口支持的最大并发
- recursion_limit：  递归调用的次数
- configurable: 运行时参数，在withhistory和graphs中常用到
### 配置传播
- 支持将配置传播到所有的子调用
- LCEL方式通过参数传播
- 函数调用或者tool方式，使用contextvars 传播

### 运行时修改配置
https://python.langchain.com/docs/how_to/configure/

## 回调 用于监控、日志
### 回调事件
Chat model start	模型启动时	on_chat_model_start
LLM start	When a llm starts	on_llm_start
LLM new token	When an llm OR chat model emits a new token	on_llm_new_token
LLM ends	When an llm OR chat model ends	on_llm_end
LLM errors	When an llm OR chat model errors	on_llm_error
Chain start	When a chain starts running	on_chain_start
Chain end	When a chain ends	on_chain_end
Chain error	When a chain errors	on_chain_error
Tool start	When a tool starts running	on_tool_start
Tool end	When a tool ends	on_tool_end
Tool error	When a tool errors	on_tool_error
Agent action	When an agent takes an action	on_agent_action
Agent finish	When an agent ends	on_agent_finish
Retriever start	When a retriever starts	on_retriever_start
Retriever end	When a retriever ends	on_retriever_end
Retriever error	When a retriever errors	on_retriever_error
Text	When arbitrary text is run	on_text
Retry	When a retry event is run	on_retry

### 回调处理
- BaseCallbackHandler
- AsyncCallbackHandler
在启动时，langchain会启动一个CallbackManager or AsyncCallbackManager 调用注册的handler
### 回调注册
- chain.invoke({"number": 25}, {"callbacks": [handler]}): 会被所有的子调用继承
- chain = TheNameOfSomeChain(callbacks=[handler])： 局部使用

## 其他
- RunnableGenerator： 在流式计算中，将用户自定义调用转换为流式
- RunnableLambda： 将函数转换为runnable

## RunnableWithMessageHistory
### invoke 流程
- RunnableBindingBase.invoke : 输入 dict
- RunnableSequence.invoke: 输入 dict
- - context.run(step.invoke, input, config, **kwargs)  step是
  - - RunnableWithMessageHistory._enter_history: RunnableLambda.invoke 输入 dict 输出：list[BaseMessage]
  - - - Runnable._call_with_config 输出：list[BaseMessage]
      - RunnableAssign.invoke
      - - RunnableParallel.invoke
      - - call_func_with_variable_args 输入 dict
  - - RunnableLambda.invoke 输入：字典和chat_history
    - - Runnable._call_with_config
      - - context.run(call_func_with_variable_args,func ...)
        - - call_func_with_variable_args
          - - RunnableLambda._invoke
            - - call_func_with_variable_args
              - - RunnableWithMessageHistory._call_runnable_sync
                - - RunnableBindingBase.invoke
                  - - RunnableBranch.invoke
                    - - branches 遍历
                      - - condition.invoke
                        - runnable.invoke
                        - self.default.invoke
                        - BaseChatPromptTemplate.format_prompt 输出ChatPromptValue

