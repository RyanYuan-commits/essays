---
type: 工具
sub-type: 绘图
finished: "true"
---
示例

```plaintext
sequenceDiagram
    participant H5 as 月卡-前端

    box 月卡后端

    participant naga as 商品服务(naga)

    participant ryukyu as 专享价服务(ryukyu)

    end

    participant rec as 推荐

    participant gin as 商品服务

    participant activity as 活动价

    participant promo as 券后价

    #

    H5->>+naga: 查询入口商品或三级页列表商品

    naga->>+ryukyu: 查询活动资格

    ryukyu->>ryukyu: 过实验, 写活动价资格

    ryukyu-->>-naga: 返回资格, 样式, 投放 ID 等

    naga->>+rec: 调用推荐出品

    rec-->>-naga: 出品 (recGoods)

    naga->>+gin: 使用 (recGoods) 中的信息调用 Gin

    gin-->>-naga: 商品详细信息 (ginGoods)

    naga-->>+activity: 调用活动价, 屏蔽 60 资格

    naga-->>activity: 查询活动库存

    activity-->>naga: 无 60 资格的活动价

    naga->>+promo: 无 60 资格的活动价调用券后价

    promo-->>-naga: 券后价

    naga->>naga: 剔除月卡相关优惠, 得到非会员价

    activity-->>-naga: 库存, 报名信息

    naga->>naga: 添加活动相关字段, 过滤非目标商品

    naga-->>-H5: 返回
```
