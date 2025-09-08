---
type: 工具
sub-type: 绘图
finished: "true"
---
```embed
title: "Mermaid"
image: "https://mermaid.nodejs.cn/favicon.svg"
description: "Create diagrams and visualizations using text and code."
url: "https://mermaid.nodejs.cn/syntax/flowchart.html"
favicon: ""
aspectRatio: "100"
```

示例:

```plaintext
---
title: 部分地区屏蔽 1399 流程
---
flowchart TD
    start(("1399 场景")) --> if1{"是否有卡?"}
    if1 --> |否| no_card[全部商品添加 exclude 列表]
    no_card --> stop((结束))
    if1 --> |是| if2[遍历商品]
    if2 --> if3{有 goodsId 或 mallId <br/> 且 <br/> 在限制名单中}
    if3 --> |是| query{默认地址在限制区域内 <br/> 或 <br/> 无默认地址}
    if3 --> |否| if4
    query --> |否| if4
    query --> |是| add_exclude[商品添加到 exclude 列表] --> if4
    if4{是否有未遍历的商品?} --> |否| stop
    if4 --> |是| if2
```