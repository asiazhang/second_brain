# BuildEngine部署导致构建中心API不可用问题分析及改进措施

---

- 日期：2018-07-31
- 作者：张恒 00368937
- 目前状态：已解决
- 摘要：BuildEngine紧急发布支持ChangeID新版本，导致构建中心API不可用40分钟
- 事故影响：部分用户构建流水线查询超时
- 解决方案：回滚增加绿区Ansible部署脚本
- 检测：CID和测试人员

待办事项：

待办事项                 					                        	    | 责任人 | 时间点
---------------------------------------------|------|----------
增加生产环境灰度部署阶段，进行验证            | 张恒 | 2018-08-02
生产环境全量部署阶段修改为手工触发             | 张恒 | 2018-08-02
新的部署脚本使用单独分支进行开发                | 张恒 | 2018-08-03
优化针对[[Ansible]]部署的回滚能力      	            | 张恒 | 2018-08-06
发布之前，在CID群里面进行公告，明确发布计划和变更范围            | 张恒 | 2018-08-02
发布窗口控制在闲时                                        | 张恒 | 2018-08-02

## 检验教训

1. 生产环境缺乏灰度部署，导致问题无法在小范围拦截
2. 生产环境缺乏自动化验收脚本，没有在部署阶段拦截问题

### 做的好得地方

### 做的不好的地方

- 使用develop分支进行绿区构建脚本的编写调试


## 时间线

2018-07-31 (UTC+8)

- `09:50` 滕汉川开发了ChangeID功能修复，需要上午发布生产环境
- `11:05` 测试环境BuildEngine部署中
- `11:09` 测试环境BuildEngine部署完成
- `11:19` 张恒批准生产环境部署
- `11:23` 生产环境BuildEngine部署完成
- `11:32` 李强发现CID构建状态一直是`waiting`
- `11:36` 李强联系张恒、滕汉川一起定位
- `11:37` 检查[[RabbitMQ]]，发现从`11:23`之后消息就没有了
- `11:39` 检查泰坦，发现构建中心部署是3天前，跟本次无关(11:20之前都是好的)
- `11:45` 定位从上一次部署到本次部署，只有2次提交
	- 增加绿区Ansible部署脚本
	- 升级jobcenter到0.3.7
- `11:55` 张恒判断应该是绿区Ansible部署脚本合并导致部署失败，对本次合并进行了Revert操作
- `11:56` 张恒登陆到某台生产环境上查看，确实是配置数据被覆盖为测试环境了，导致无法查询
- `11:59` 测试环境开始重新部署
- `12:02` 生产环境开始重新部署
- `12:06` 生产环境部署完成，问题解决


### Sentry捕捉到的异常

版本0.3.7对应的应该是生产环境`production`，而不是测试环境`develop`。这个错误就是生产环境配置数据被覆盖为测试环境导致的。

### Ansible group vars导致配置数据失败

为了简化黄/绿区部署脚本，部署脚本优化里面提取了公共变量，但是在生产环境中覆盖没有生效，导致部署后属性被替换为测试环境的值。