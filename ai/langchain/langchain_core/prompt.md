# prompt
![runnable](./classes_prompts.png)

## 运转机制
### BaseMessagePromptTemplate示例
```
class BaseMessagePromptTemplate(Serializable, ABC):
    @abstractmethod
    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
    @property
    @abstractmethod
    def input_variables(self) -> list[str]:

class _StringImageMessagePromptTemplate(BaseMessagePromptTemplate):
    prompt: Union[
        StringPromptTemplate, list[Union[StringPromptTemplate, ImagePromptTemplate]]
    ]
    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        实现
    def input_variables(self) -> list[str]:
        实现


class HumanMessagePromptTemplate(_StringImageMessagePromptTemplate):

class BasePromptTemplate(
    RunnableSerializable[dict, PromptValue], Generic[FormatOutputType], ABC
):
    @abstractmethod
    def format_prompt(self, **kwargs: Any) -> PromptValue:
    def invoke(
        self, input: dict, config: Optional[RunnableConfig] = None, **kwargs: Any
    ) -> PromptValue:
    @abstractmethod
    def format(self, **kwargs: Any) -> FormatOutputType:
class StringPromptTemplate(BasePromptTemplate, ABC):
    def format_prompt(self, **kwargs: Any) -> PromptValue:
        实现

class PromptTemplate(StringPromptTemplate):
    def format(self, **kwargs: Any) -> str:
        实现

class PromptValue(Serializable, ABC):
    @abstractmethod
    def to_string(self) -> str:

    @abstractmethod
    def to_messages(self) -> list[BaseMessage]:

class StringPromptValue(PromptValue):
    def to_string(self) -> str:
        """Return prompt as string."""
        return self.text

    def to_messages(self) -> list[BaseMessage]:
        """Return prompt as messages."""
        return [HumanMessage(content=self.text)]
```

### 调用过程
- HumanMessagePromptTemplate.from_template
- - PromptTemplate.from_template填充_StringImageMessagePromptTemplate.prompt
- HumanMessagePromptTemplate.format_messages
- - _StringImageMessagePromptTemplate.format
  - - PromptTemplate.format 使用字典填充变量构成字符串
    - 构建BasicMessage并返回
- PromptTemplate.from_template
- - 提取input_variables
  - 构建PromptTemplate 类
- PromptTemplate.invoke
- - BasePromptTemplate.invoke
  - - Runnable._call_with_config
    - - StringPromptTemplate.format_prompt
      - PromptTemplate.format
      - return StringPromptValue

