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
```

### 调用过程
- HumanMessagePromptTemplate.from_template
- - PromptTemplate.from_template填充_StringImageMessagePromptTemplate.prompt

