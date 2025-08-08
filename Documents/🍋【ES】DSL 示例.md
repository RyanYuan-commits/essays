![[DSL 查询思维导图.png|900]]
## 单独查询
### (1) 基本查询
- 查询指定字段中匹配关键词或短语的文档
- 示例：查询 "title" 字段中包含 "Elasticsearch" 的文档
```http
GET /index/_search
{
"query": {
		"match": {
			"title": "Elasticsearch"
		}
	}
}
```
### (2) 多字段查询
- 在多个字段中匹配指定的关键词或短语
- 示例：查询 "title" 和 "content" 字段中匹配 "ElasticSearch" 的文档
```http
GET /index/_search
{
	"query": {
		"multi_match": {
			"query": "Elasticsearch",
			"fields": ["title", "content"]
		}
	}
}
```
### (3) 范围查询
- 根据范围条件匹配字段中的值
- 示例：查询价格字段在 50 到 100 之间的文档
```http
GET /index/_search
{
	"query": {
		"range": {
			"price": {
				"gte": 50,
				"lte": 100
			}
		}
	}
}
```
### (4) 布尔查询
- 将多个查询条件组合成逻辑上的 AND、OR 或者 NOT 关系
- 示例：查询标题包含 “ElasticSearch" 且价格大于 50 的文档
```http
GET /index/_search
{
	"query": {
		"bool": {
		"must": [
				{ "match": { "title": "Elasticsearch" } },
				{ "range": { "price": { "gte": 50 } } }
			]
		}
	}
}
```
### (5) 聚合查询
- 计算和统计数据集中的汇总信息
- 示例：计算你字段 "sales" 的总和作为结果返回
```http
GET /index/_search
{
	"aggs": {
	"total_sales": {
			"sum": { "field": "sales" }
		}
	}
}
```
### (6) 排序与分页
- 示例：按照"timestamp"字段的降序对结果进行排序。
```http
GET /index/_search
{
	"sort": [
		{ "timestamp": { "order": "desc" } }
	]
}
```
- 示例：返回从0开始的10个文档作为结果。
```http
GET /index/_search
{
	"from": 0,
	"size": 10,
	"query": {
		"match_all": {}
	}
}
```

## 组合查询
### 组合多个 must 查询
```http
GET /index/_search
{
	"query": {
		"bool": {
			"must": [
				{ "match": { "title": "Elasticsearch" } },
				{ "match": { "content": "数据分析" } }
			]
		}
	}
}
```
### 组合 must 和 should 查询
- 示例：要求标题包含"Elasticsearch"且（价格大于等于50或评分高于4）的文档。
```http
GET /index/_search
{
	"query": {
		"bool": {
			"must": [
				{ "match": { "title": "Elasticsearch" } }
			],
			"should": [
				{ "range": { "price": { "gte": 50 } } },
				{ "range": { "rating": { "gt": 4 } } }
			]
		}
	}
}
```
