## 管理索引
### 查看所有的索引
```http
GET _cat/indices?v
```
返回结果：
![[ES 查看所有索引.png]]
### 删除索引
```http
DELETE /skuinfo
```
### 新增索引
```http
PUT /user
```
### 创建映射
类似于关系型数据库中的表
```http
PUT /user/_mapping
{
	"properties": {
		"name": {
			"type": "text",
			"analyzer": "ik_smart",
			"search_analyzer": "ik_smart"
		},
		"city": {
			"type": "text",
			"analyzer": "ik_smart",
			"search_analyzer": "ik_smart"
		},
		"age": {
			"type": "long"
		},
		"description": {
			"type": "text",
			"analyzer": "ik_smart",
			"search_analyzer": "ik_smart"
		}
	}
}
```
### 新增文档数据
```http
PUT /user/_doc/1
{
  "name": "李四",
  "age": 22,
  "city": "深圳",
  "description": "李四来自湖北武汉!"
}
```
再新增一些其他数据
```http
# 新增文档数据 id=2
PUT /user/_doc/2
{
  "name":"王五",
  "age":35,
  "city":"深圳",
  "description":"王五家住在深圳！"
}
# 新增文档数据 id=3
PUT /user/_doc/3
{
  "name":"张三",
  "age":19,
  "city":"深圳",
  "description":"在深圳打工，来自湖北武汉"
}
# 新增文档数据 id=4
PUT /user/_doc/4
{
	"name":"张三丰",
	"age":66,
	"city":"武汉",
	"description":"在武汉读书，家在武汉！"
}
# 新增文档数据 id=5
PUT /user/_doc/5
{
	"name":"赵子龙",
	"age":77,
	"city":"广州",
	"description":"赵子龙来自深圳宝安，但是在广州工作！",
	"address":"广东省茂名市"
}
# 新增文档数据 id=6
PUT /user/_doc/6
{
	"name":"赵毅",
	"age":55,
	"city":"广州",
	"description":"赵毅来自广州白云区，从事电子商务8年！"
}
# 新增文档数据 id=7
PUT /user/_doc/7
{
	"name":"赵哈哈",
	"age":57,
	"city":"武汉",
	"description":"武汉赵哈哈，在深圳打工已有半年了，月薪7500！"
}
```
### 修改数据
#### 使用 PUT 替换数据
使用下面的方式会将存在的 doc执行 **替换** 操作
```http
#更新数据,id=4
PUT /user/_doc/4
{
	"name":"张三丰",
	"description":"在武汉读书，家在武汉！在深圳工作！"
}
```
使用 GET 命令查看
```http
GET /user/_doc/4
```
结果如下：
```json
{
  "_index" : "user",
  "_type" : "_doc",
  "_id" : "4",
  "_version" : 2,
  "_seq_no" : 8,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "张三丰",
    "description" : "在武汉读书，家在武汉！在深圳工作！"
  }
}
```
#### 使用 POST 来更新数据：
```http
#更新数据,id=4
POST /user/_update/4
{
  "doc": {
    	"name":"张三丰",
	    "description":"在武汉读书，家在武汉！在深圳工作！"
  }
}
```
结果如下：
```json
{
  "_index" : "user",
  "_type" : "_doc",
  "_id" : "4",
  "_version" : 11,
  "_seq_no" : 17,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "张三丰",
    "age" : 66,
    "city" : "武汉",
    "description" : "在武汉读书，家在武汉！在深圳工作！"
  }
}
```
### 删除 Document
```http
#删除数据
DELETE user/_doc/7
```
## 数据查询
### 查询所有数据
```http
GET /user/_search
```
### 根据 ID 查询
```http
GET /user/_doc/2
```
### Sort 排序
```http
GET /user/_search
{
	"query": {
		"match_all": {}
	},
	"sort": {
		"age": {
			"order": "desc"
		}
	}
}
```
### 分页
```http
GET /user/_search
{
	"query":{
		"match_all": {}
	},
	"sort":{
		"age":{
			"order":"desc"
		}
	},
	"from": 0,
	"size": 2
}
```
- from 从下 N 条记录开始查询
- size：每页显示的条数
## 查询模式
### term 查询
term 主要用于分词精确匹配，如字符串、数值、日期等
```http
GET /user/_search
{
  "query": {
    "term": {
      "city": "武汉"
    }
  }
}
```
### terms 查询
terms 允许指定多个匹配条件，如果一个字段指定了多个值，那文档需要一起去做匹配。
```http
GET /user/_search
{
	"query": {
		"terms": {
			"city": [ "武汉", "广州" ]
		}
	}
}
```
### match 查询
```http
GET _search
{
	"query": {
		"match": {
			"city": "广州武汉"
		}
	}
}
```
### query_string 查询
```http
GET _search
{
	"query": {
		"query_string": {
			"default_field": "city",
			"query": "广州武汉"
		}
	}
}
```
### range 查询
```http
GET /user/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 30,
        "lte": 50
      }
    }
  }
}
```
### exists 查询
可以用于查找拥有某个域的数据
```http
GET /user/_search 
{
  "query": {
    "exists": {
      "field": "address"
    }
  }
}
```
### boolean 查询
bool 可以用来合并多个条件查询结果的布尔逻辑，它包含以下操作符：
	must : 多个查询条件的完全匹配,相当于 and。
	must_not : 多个查询条件的相反匹配，相当于 not。
	should : 至少有一个查询条件匹配, 相当于 or。
这些参数可以分别继承一个过滤条件或者一个过滤条件的数组：
```http
GET _search
{
	"query": {
		"bool": {
			"must": [
				{
					"term": {
						"city": {
							"value": "深圳"
						}
					}
				},
				{
					"range":{
						"age":{
							"gte":20,
							"lte":99
						}
					}
				}
			]
		}
	}
}
```
### match_all 查询
可以查询到所有文档，是没有查询条件下的默认语句。
```http
#查询所有 match_all
GET _search
{
	"query": {
		"match_all": {}
	}
}
```
### match 查询
match查询是一个标准查询，不管你需要全文本查询还是精确查询基本上都要用到它。如果你使用 match 查询一个全文本字段，它会在真正查询之前用分析器先分析match一下查询字符：
```http
#字符串匹配
GET _search
{
	"query": {
		"match": {
			"description": "武汉广州"
		}
	}
}
```
### prefix 查询
以什么字符开头的，可以更简单地用 prefix ,例如查询所有以张开始的用户描述
```http
#前缀匹配 prefix
GET _search
{
	"query": {
		"prefix": {
			"name": {
				"value": "赵"
			}
		}
	}
}
```
### multi_match 查询
multi_match查询允许你做match查询的基础上同时搜索多个字段，在多个字段中同时查一个
```http
#多个域匹配搜索
GET _search
{
	"query": {
		"multi_match": {
			"query": "深圳",
			"fields": [
				"city",
				"description"
			]
		}
	}
}
```
### filter
因为过滤可以使用缓存，同时不计算分数，通常的规则是，使用查询（query）语句来进行 全文 搜索或者其它任何需要影响 相关性得分 的搜索。除此以外的情况都使用过滤（filters)
```http
GET /user/_search
{
	"query": {
		"bool": {
			"filter": {
				"range":{
					"age":{
						"gte":25,
						"lte": 80
					}
				}
			}
		}
	}
}
```