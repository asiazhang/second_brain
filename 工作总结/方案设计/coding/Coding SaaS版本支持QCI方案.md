# Coding SaaS版本支持QCI方案（基座部分）

## 需求来源
Coding SaaS版本之前采用[[jenkins]]作为CI构建引擎，后续想把OA版本的CodingCI也进行接入，在SaaS版本中支持2套引擎。

## 当前问题

1. CodingCI当前只支持OA版本基座，跟SaaS版本的基座接口有差异
2. CodingCI版本基座的接口，跟SaaS版本基座接口也有差异

## 解决方案

### 前期

根据当前基座的实现标准化proto文件。OA版本无需增加Adapter（因为代码都是转发，无太大意义）。SaaS版本增加QCIAdapter，并依赖于proto标准化的SDK。此SDK作为QCI接入的标准，SaaS版本QCIAdapter实现以下功能：

1. 如果SaaS版本已有实现，那么直接转发请求，并将参数/返回值标准化。
2. 如果SaaS版本没有实现接口，那么QCIAdapter直接实现，后续逐步交由SaaS同学合并到版本中。

见下图：

![[coding_saas_adapter_early.png]]

### 中期

部分接口已经由SaaS同学实现，Adapter纯转发。

![[coding_saas_adapter_middle.png]]

### 后期

OA版本和SaaS版本的接口都已经根据proto标准化，去掉Adapter。

![[coding_saas_adapter_after.png]]

## 如何保证接口升级稳定性

由于当前是通过适配器模式进行转发，那么一个无可避免的问题就是OA或者SaaS版本的接口变化可能会导致Adapter失效，需要某种自动化的验证方式来保证依赖稳定性。我们考虑使用契约验证的方式，在部署流水线上增加阶段来验证。

![[coding_saas_qci_test.png]]

## 接口差异和标准化

经过统计和与SaaS版本接口差异分析，情况如下：

- 无差异接口：8个
- 参数差异接口：7个
- SaaS版本缺失接口：21个
- 暂时无需提供接口：4个

SaaS版本接口缺失有点多，需要进行接口补全。