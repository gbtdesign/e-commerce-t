---
kind: external_dependency
name: RabbitMQ 消息队列（削峰/异步下单）
slug: rabbitmq
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
---

本项目通过 `github.com/streadway/amqp` 客户端接入 RabbitMQ，使用简单模式（Simple Direct Queue）实现下单削峰：
- 生产者（fronted/web/controllers/product_controller.go）将 `userID + productID` 封装为 JSON 消息，经 `PublishSimple` 投递到队列名 `imoocProduct`。
- 消费者侧设置了 QoS prefetch=1，保证单条处理完成后再拉下一条。