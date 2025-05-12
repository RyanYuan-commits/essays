GitHub：https://github.com/medcl/elasticsearch-analysis-ik/releases
IK 分词器是目前使用最广泛的中文分词器，提供了多种分词方式。
### 1 安装方式
```bash
# 将 8.4.1 切换为当前 ElasticSearch 的版本
bin/elasticsearch-plugin install https://get.infini.cloud/elasticsearch/analysis-ik/8.4.1
```
### 2 分词模式
#### 2.1 最细粒度拆分 - ik_max_word
将文本做最细粒度的拆分，比如会将“中华人民共和国人民大会堂”
拆分为“中华人民共和国、中华人民、中华、华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。
```http
POST /_analyze
{
  "text":"中华人民共和国人民大会堂",
  "analyzer":"ik_max_word"
}
```
结果如下：
```json
{
  "tokens" : [
    {
      "token" : "中华人民共和国",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "中华人民",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "中华",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "华人",
      "start_offset" : 1,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "人民共和国",
      "start_offset" : 2,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 4
    },
    {
      "token" : "人民",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 5
    },
    {
      "token" : "共和国",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 6
    },
    {
      "token" : "共和",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 7
    },
    {
      "token" : "国人",
      "start_offset" : 6,
      "end_offset" : 8,
      "type" : "CN_WORD",
      "position" : 8
    },
    {
      "token" : "人民大会堂",
      "start_offset" : 7,
      "end_offset" : 12,
      "type" : "CN_WORD",
      "position" : 9
    },
    {
      "token" : "人民大会",
      "start_offset" : 7,
      "end_offset" : 11,
      "type" : "CN_WORD",
      "position" : 10
    },
    {
      "token" : "人民",
      "start_offset" : 7,
      "end_offset" : 9,
      "type" : "CN_WORD",
      "position" : 11
    },
    {
      "token" : "大会堂",
      "start_offset" : 9,
      "end_offset" : 12,
      "type" : "CN_WORD",
      "position" : 12
    },
    {
      "token" : "大会",
      "start_offset" : 9,
      "end_offset" : 11,
      "type" : "CN_WORD",
      "position" : 13
    },
    {
      "token" : "会堂",
      "start_offset" : 10,
      "end_offset" : 12,
      "type" : "CN_WORD",
      "position" : 14
    }
  ]
}
```
#### 2.2 最粗粒度拆分 - ik_smart
将文本做最粗粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂。
```http
POST /_analyze
{
  "text": "中华人民共和国人民大会堂",
  "analyzer": "ik_smart"
}
```
结果如下：
```json
{
  "tokens" : [
    {
      "token" : "中华人民共和国",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "人民大会堂",
      "start_offset" : 7,
      "end_offset" : 12,
      "type" : "CN_WORD",
      "position" : 1
    }
  ]
}
```
### 3 自定义扩展字典
IK分词器的两种模式的最佳实践是：索引时用 ik_max_word，搜索时用 ik_smart，索引时最大化的将文章内容分词，搜索时更精确的搜索到想要的结果。
>[!example] 举个例子
用户输入“华为手机”搜索，此时应该搜索出“华为手机”的商品，而不是“华为”和“手机”这两个词，这样会搜索出华为其它的商品，此时使用 ik_smart 和 ik_max_word 都会将华为手机拆分为华为和手机两个词，那些只包括“华为”这个词的信息也被搜索出来了，我的目标是搜索只包含华为手机这个词的信息，这没有满足我的目标。

ik_smart默认情况下分词“华为手机”，依然会分成两个词“华为”和“手机”，这时需要使用自定义扩展字典。
```bash
# 进入 ES 容器
docker exec -it elasticsearch /bin/sh

# 增加自定义字典文件
cd plugins/ik/config/
vi new_word.dic

# 修改配置文件
vim IKAnalyzer.cfg.xml

# 重新运行容器
docker restart elasticsearch
```
配置文件修改内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">new_word.dic</entry>
	<!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key=
	"remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```