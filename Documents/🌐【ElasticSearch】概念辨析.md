### 1 简单介绍
#### 1.1 ELK Stack 工具集
Elastic Stack，以前也被称为 ELK Stack，是一套开源的、用于数据收集、存储、分析和可视化的工具集，由 Elasticsearch、Logstash、Kibana 和 Beats 等主要组件组成。
![[ELK Stack 组成.png|700]]
#### 1.2 ElasticSearch
ElasticSearch 底层是基于 Lucene 来实现的，**Lucene** 是一个Java语言的搜索引擎类库，是Apache公司的顶级项目，由 DougCutting 于 1999 年研发，官网地址：[https://lucene.apache.org/](https://lucene.apache.org/) 。
![[Lucene.png|center|700]]
**elasticsearch**的发展历史：
- 2004年Shay Banon基于Lucene开发了Compass
- 2010年Shay Banon 重写了Compass，取名为Elasticsearch。
![[ES 图片.png|center|700]]
### 2 倒排索引
#### 2.1 正向索引 vs 倒排索引
与倒排索引相对的是正排索引，MySQL 中的索引就是一个正排索引，以一本书来举例，正排索引类似目录，目录项中的页码相当于索引 ID，对应的是具体文章内容：
![[正排索引案例.png|350]]
而倒排索引类似索引页，描述的是 **知识点** 和页码之间的关系：
![[倒排索引案例.png|350]]
比如我们想要查询有 ElasticSearch 这个知识点的文章，首先通过倒排索引查询到其在 ID 为 1 和 3 的文章中出现过；再根据这个 ID 去正排索引就可以查询到完整的内容。
#### 2.2 详细解释
ES 中存储数据的基本单位是文档 Document，具体来说是一个 JSON 格式的数据，其中包含多个 **字段**，每个字段都有自己的倒排索引 —— **倒排索引是加在文档字段上的**。
我们不可能按照字去分割文章，需要将文本转化为一系列的词项，这个过程就叫做分词，处理这个过程的组件叫做分词器。
![[倒排索引表格.png|center|450]]
上面展示的就是一个简单的倒排索引的示例，由三个关键部分组成：
- `Term Dictionary`：词项字典，存储 **倒排索引** 中所有唯一词项（Terms）及其相关元数据的一个有序集合。
- `Term Index`：词项索引，用于快速的定位某个词在词项字典中的位置。
- `Posting List`：与某个词项相关的文档 ID 列表，表示该词项在哪些文档中出现，可能包含附加信息（如词频、位置）。
当用户搜索一个词项时，首先会在词项索引中找到该词项在词项字典中的位置，就可以得到这个词项对应的 **Post List**；最终，根据 **Post List** 就可以获取相关文档。
### 3 分词器
#### 3.1 基本组成
分词器是 ES 中专门处理分词的组件，英文为 Analyzer，它的组成如下：
- Character Filters：针对原始文本进行处理，比如去除HTML特殊标记
- Tokenizer：将原始文本按照一定规则分为单词
- Token Filters：针对tokenizer处理的单词再加工，比如转小写、删除或新增等处理
#### 3.2 默认分词器
##### 3.2.1 对英文的分词
使用 `_analyze` 请求可以查看分词的具体结果
```http
POST /_analyze
{
	"text": "You can use Elasticsearch to store, search, and manage data",
	"analyzer": "standard"
}
```
结果如下：
```json
{
  "tokens" : [
    {
      "token" : "you",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "can",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "use",
      "start_offset" : 8,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "elasticsearch",
      "start_offset" : 12,
      "end_offset" : 25,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "to",
      "start_offset" : 26,
      "end_offset" : 28,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "store",
      "start_offset" : 29,
      "end_offset" : 34,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "search",
      "start_offset" : 36,
      "end_offset" : 42,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "and",
      "start_offset" : 44,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "manage",
      "start_offset" : 48,
      "end_offset" : 54,
      "type" : "<ALPHANUM>",
      "position" : 8
    },
    {
      "token" : "data",
      "start_offset" : 55,
      "end_offset" : 59,
      "type" : "<ALPHANUM>",
      "position" : 9
    }
  ]
}
```
会将英文按照单词进行分割。
##### 3.2.2 对中文的分词
同样，使用 `_analyze` 来查看一下对中文的分词：
```http
POST /_analyze
{
	"text": "中华人民共和国人民大会堂",
	"analyzer": "standard"
}
```
结果如下：
```json
{
  "tokens" : [
    {
      "token" : "中",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "华",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "人",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "民",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "共",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    },
    {
      "token" : "和",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "<IDEOGRAPHIC>",
      "position" : 5
    },
    {
      "token" : "国",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "<IDEOGRAPHIC>",
      "position" : 6
    },
    {
      "token" : "人",
      "start_offset" : 7,
      "end_offset" : 8,
      "type" : "<IDEOGRAPHIC>",
      "position" : 7
    },
    {
      "token" : "民",
      "start_offset" : 8,
      "end_offset" : 9,
      "type" : "<IDEOGRAPHIC>",
      "position" : 8
    },
    {
      "token" : "大",
      "start_offset" : 9,
      "end_offset" : 10,
      "type" : "<IDEOGRAPHIC>",
      "position" : 9
    },
    {
      "token" : "会",
      "start_offset" : 10,
      "end_offset" : 11,
      "type" : "<IDEOGRAPHIC>",
      "position" : 10
    },
    {
      "token" : "堂",
      "start_offset" : 11,
      "end_offset" : 12,
      "type" : "<IDEOGRAPHIC>",
      "position" : 11
    }
  ]
}
```
#### 3.3 中文分词
默认分词器对中文的分词支持并不友好，所以我们需要安装额外的中文分词器来更好的支持对中文的索引。
不同于英文的是，中文句子中没有词的界限，因此在进行中文自然语言处理时，通常需要先进行分词，分词效果将直接影响词性、句法树等模块的效果；当然分词只是一个工具，场景不同，要求也不同。
常见的中文分词工具：
```markdown
[中科院计算所NLPIR](http://ictclas.nlpir.org/nlpir/)  
[ansj分词器](https://github.com/NLPchina/ansj_seg)  
[哈工大的LTP](https://github.com/HIT-SCIR/ltp)  
[清华大学THULAC](https://github.com/thunlp/THULAC)  
[斯坦福分词器](https://nlp.stanford.edu/software/segmenter.shtml)  
[Hanlp分词器](https://github.com/hankcs/HanLP)  
[结巴分词](https://github.com/yanyiwu/cppjieba)  
[KCWS分词器(字嵌入+Bi-LSTM+CRF)](https://github.com/koth/kcws)  
[ZPar](https://github.com/frcchang/zpar/releases)
```