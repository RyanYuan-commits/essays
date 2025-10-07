---
title: 如果您使用得足够错误，任何东西都可能成为消息队列
source: https://xeiaso.net/blog/anything-message-queue/
created: 2025-08-29
description: 本文以讽刺的方式，通过在AWS S3上构建一个基于TUN/TAP设备的IPv6网络，展示了如何“错误地”将对象存储服务当作消息队列使用，以规避NAT Gateway的高昂流量费用，尽管其自身运作成本极其不切实际。
finished: "false"
tag: clipper
cover: https://stickers.xeiaso.net/sticker/cadey/coffee
updated: 2025-09-27 22:34:06
---
关键词：AWS, S3, IPv6, TUN/TAP, 消息队列, NAT Gateway, 成本分析, Hoshino, satire, Go, Terraform, 网络协议, 云服务, Tailscale

**详细总结：**

本文以讽刺的口吻探讨了在AWS云环境中，如何“错误地”使用S3服务来规避昂贵的托管NAT Gateway（NAT网关）费用。作者提出，Managed NAT Gateway是导致初创公司云账单高昂的祸害，并建议使用Tailscale出口节点作为生产环境中更合理的替代方案（这是文中唯一推荐的实际解决方案）。

文章的核心是一个名为“Hoshino”（星野）的“被诅咒的”概念实现。其基本原理是：
1.  **S3是云的`malloc()`**：S3可以存储任意字节数据，类似于C语言中的动态内存分配。
2.  **IPv6数据包是字节**：网络数据包本质上就是字节序列。
3.  **TUN设备**：Linux的TUN/TAP设备允许用户空间应用程序控制网络数据包的读写。

Hoshino系统将这些概念结合起来，通过以下方式“构建”一个基于S3的IPv6网络：
*   在节点上创建并配置一个TUN设备（`hoshino0`），使其只处理IPv6流量。
*   每个节点的IPv6地址根据系统`machine-id`生成。
*   Hoshino读取从内核通过TUN设备发出的IPv6数据包，并将其作为S3对象上传，S3对象的键值结合了目标IPv6地址和ULID（确保排序）。
*   另一个循环不断从S3读取这些数据包，然后通过TUN设备写回目标节点的内核。
*   利用`cardio`（一个Go语言的心跳工具）动态调整轮询S3的频率，以应对流量变化。
*   使用Terraform配置IAM权限和S3存储桶的生命周期策略（对象一天后自动删除）。

作者在亲身实践后，成功地实现了通过S3发送ping和TCP数据包，并展示了`curl`获取指标页面的输出，证明了这种“可怕地”工作方式的可行性。然而，随后的成本分析揭示，尽管可以避免NAT Gateway的流量费用，但由于频繁的S3操作（每毫秒一次的轮询，即使在空闲状态下，每个节点每天也要产生大量GET和PUT请求），Hoshino的实际运行成本远高于正常的网络传输成本（每GB 0.07美元）。

文章最后提到，由于其性质，作者需要与雇主（Tailscale）协商知识产权，并决定暂时不开源这个“暴行”，尽管它在技术上证明了“一切皆可作消息队列”的荒谬性。