>[!tips] 为什么需要别名？
在 mysql 中 我们经常遇到产品修改需求 我们可能会在原有数据库表基础上 对字段 索引 类型进行修改；比如增加一个字段、添加一个字段的索引又或者修改某个字段的类型，这一切都看起来这么自然；不过在 ES 是行不通的 ES 的 mapping 一旦设置了之后，可以改，但是改了没有用，因为ES默认是对所有字段进行索引 如果你修改了mapping 那么已经索引过的数据就必须全部重新索引一遍 ,ES 没有提供这个机制, 只能利用别名手工刷数据。

### 添加索引别名
```http
PUT article1/_alias/article
{
	"acknowledged" : true
}
```
### 创建新索引 artucle2
添加了一个 owner 字段
```http
PUT article2
{
	"settings": {
		"number_of_shards": 3,
		"number_of_replicas": 1 ,
		"analysis" : {
			"analyzer" : {
				"ik" : {
					"tokenizer" : "ik_max_word"
				}
			}
		}
	},
	"mappings": {
		"properties": {
			"id": {
				"type": "keyword"
			},
			"title": {
				"type": "text",
				"analyzer": "ik_max_word"
			},
			"content": {
				"type": "text",
				"analyzer": "ik_max_word"
			},
			"viewCount": {
				"type": "integer"
			},
			"creatDate": {
				"type": "date",
				"format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
			},
			"tags": {
				"type": "keyword"
			},
			"owner": {
				"type": "keyword"
			}
		}
	}
}
```
### 重建索引
```http
PUT _reindex
{
	"source": {
		"index": "article1"
	},
	"dest": {
		"index": "article2"
	}
}
```
### 修改别名映射
```http
POST /_aliases
{
	"actions": [
		{
			"remove": {
				"index": "article1",
				"alias": "article"
			},
			"add": {
				"index": "article2",
				"alias": "article"
			}
		}
	]
}
```
### 使用别名搜索
```http
GET /article/_search
{
	"query": {
		"match_all": {}
	}
}
```