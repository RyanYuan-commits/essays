![[首页头图.png]]

## 📆 今天是  ==`=dateformat(date(today),"yyyy 年 M 月 d 日")`==

**一切新的东西都是从艰苦斗争中锻炼出来的...**

````col 
```col-md 
>[!note] 日记
>```dataview
>LIST FROM "" WHERE type = "日记" and file.name != "🍒 Diary-Template"  SORT file.ctime desc limit 7
>```
``` 
```col-md 
>[!note] 周报
>```dataview
>LIST FROM "" WHERE type = "周报" SORT file.ctime desc
>```
``` 
````

## ✒️ 写作
```dataview
table without id "🕹️" + type as "🎈类型", file.link as "🗃️文件", dateformat(file.mtime, "⏰yyyy.MM.dd HH:mm:ss") as "⌛修改时间"
from ""
where finished != null and finished != "true" and type != null and type != "周报"
sort file.mtime desc
limit 5
```

## 🚪 开始探索

````col 
```col-md 
### 💻 编程语言
- [[🦴【Java】.canvas|🦴【Java】]]
- [[🦝【Python】.canvas|🦝【Python】]]
- [[🥖【Golang】.canvas|🥖【Golang】]]
- [[🎮【FrontEnd】.canvas|🎮【FrontEnd】]]
``` 

```col-md 
### 🥅 中间件
- [[🍋 MySQL 专题.canvas|🍋 MySQL 专题]]
- [[🥠 Redis 专题.canvas|🥠 Redis 专题]]
- [[🌼 ElasticSearch 专题.canvas|🌼 ElasticSearch 专题]]
- [[🍒 消息队列专题.canvas|🍒 消息队列专题]]
``` 

```col-md 
### 🚇 编程地基
- [[🚆 计算机网络.canvas|🚆 计算机网络]]
- [[🗽 操作系统.canvas|🗽 操作系统]]
- [[🏜️ 分布式.canvas|🏜️ 分布式]]
- [[🎨 系统设计.canvas|🎨 系统设计]]
``` 
````
