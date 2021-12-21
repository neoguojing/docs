# Pyresparser 简历解析
## 安装
- pip install pyresparser

## spacy 自然语言处理工具
- conda install -c conda-forge spacy
- pip install -U spacy
- python -m spacy download en_core_web_sm //英文模型
- python -m spacy download zh_core_web_sm //中文模型

### 分词 tokenize
- 将句子分解为词语，长句子拆成有意义的小部件
### 词干化 lemmatize
- 将复数，动词的时态等去掉，英文
### 词性标注 POS
- 标注词语是动词，名词等
### 命名实体识别（NER） 
- 命名实体识别的任务就是识别出待处理文本中三大类（实体类、时间类和数字类）、七小类（人名、机构名、地名、时间、日期、货币和百分比）命名实体
### 名词短语提取

## nltk 自然语义处理库
- python -m nltk.downloader words
- python -m nltk.downloader stopwords
### 
