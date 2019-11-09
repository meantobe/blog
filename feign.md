---
title: Spring Cloud OpenFeign的两个问题
date: 2019-11-09 17:19:35
tags:
- Spring Cloud
- Feign
---

摘要：
1. Spring Cloud OpenFeign自Greenwich起支持query map声明为model。
2. Spring Cloud OpenFeign的Greenwich.SR2修复+号在query string中未encode问题。

<!-- more -->
# Feign与OpenFeign

[Feign](https://github.com/OpenFeign/feign)是Netflix开源的一个声明式HTTP客户端，9.0起迁移至OpenFeign，简要的历史可见 [Move to a new org](https://github.com/OpenFeign/feign/issues/373) 。

Spring Cloud中和Feign相关的代码原在Spring Cloud Netflix项目内。在Finchley.M7（Spring Cloud Netflix 2.0.0.M7）版本中，Feign相关的代码迁移到一个新项目Spring Cloud OpenFeign内。即`org.springframework.cloud.netflix.feign`相关代码改为`org.springframework.cloud.openfeign`。

# Spring OpenFeign的两个问题

## Model作为Query Map的支持
Feign支持Model转换为QueryMap（9.7.0之后可以设置encode: https://github.com/OpenFeign/feign/pull/667 ），但是Feign的`@QueryMap`注解由于缺少`value`字段，在Spring Cloud OpenFeign中并不支持。

Spring Cloud OpenFeign的2.1.x版本中增加`@SpringQueryMap`注解。

## Feign对Query String中特殊字符的encode
Feign 10.x中出现一个新问题：对query string未将+号转换为%2B，在10.2.0版本中修复（[Adding URI segment specific encoding](https://github.com/OpenFeign/feign/pull/882)）。

Spring Cloud OpenFeign 2.1.2版本升级到Feign 10.3.0，修复该问题。

总结：目前（2019.11.09）使用Spring Cloud Greenwich.SR2可以避免以上两个已知问题。