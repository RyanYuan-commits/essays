![[首页头图.png|1800]]

## 📆 今天是 ==`=dateformat(date(today),"yyyy 年 M 月 d 日")`==

```dataviewjs
let start = dv.page("统计总览")?.start_date ?? "2024-08-01";
let days = moment().diff(moment(start), "days") + 1;
dv.span("你已使用 Obsidian " + days + " 天, 创建笔记 " + dv.pages().length + " 篇");
```

**人们总是高估一天之内能做的事, 低估一年之内能走多远...**

不钻细节 : 只看流程; 不看过程 : 只看结论; 再看细节 :再看过程

---

## ✒️ 最近关注

![[写作.base]]

---

## 🚪 开始探索

![[Gather视图.base]]

---

## 🔅 稍后读

![[稍后读.base]]
