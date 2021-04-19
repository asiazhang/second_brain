# 总结CI接入SaaS流程（基座篇）

QCI接入SaaS在基座部分总体分为以下几个阶段：

## 1. 整理CI使用基座接口

分析当前CI使用的基座接口（包含gRPC和Http），整理出对应Excel表格

> **确保**：
> 1. 尽可能将所有使用接口全部包含，不要遗漏，否则后续联调进度会受影响

## 2. 整理接口差异表

联系SaaS的同学，根据此表格分析接口差异，不同场景分别处理：

> **请认真整理接口差异表，这决定了后续适配层的开发工作量。**

### 2.1 完全兼容场景

哪些接口已经实现并完全兼容，这些无需改动，直接转发即可。

> 实际来看此场景较少。

### 2.2 接口差异场景

 哪些接口已有，但是跟OA版本这边存在差异（传入参数/传出参数）
1. 返回中缺少某些属性（80%场景，**重点关注缺少的属性CI是否需要，逐个接口找CI的同学确认**）
2. 调用端缺少某些字段
3. 属性名称差异
4. 属性格式差异（比如时间戳格式）
5. 接口名称差异

这里唯一需要SaaS同学处理的场景为属性缺失，适配层无法解决，通过saas已有接口也搞不定，必须saas的gRPC接口修改。

### 2.3 接口缺失场景

 哪些接口没有，又分为以下情况：
1. SaaS版本没有，但是适配层可以通过SaaS其他接口间接实现
2. 必须依赖SaaS版本实现

> 场景二请联系Saas的同学提需求。

### 2.4 接口冲突场景

某些接口OA版本跟SaaS版本都有，完全重名，而且返回格式有差异。这种情况适配层无法直接处理，需要将原有OA版本接口稍作修改，举例说明如下：

oa版本和saas版本的`Credential.proto`文件接口`CredentialService.getById`返回值冲突，无法兼容，修改如下：

1. 将OA版本`Credential.proto`文件修改为`CredentialOA.proto`
2. 将`CredentialOA.proto`文件中包名修改为`app.grpc.enterprise.credential.oa`，避免跟saas版本冲突
3. 将`CredentialOA.proto`文件中服务名称`CredentialService`修改为`CredentialServiceOA`，避免跟saas版本冲突

这样CI需要做小调整，根据当前是否是SaaS场景来判断是调用`CredentialService.getById`还是`CredentialServiceOA.getById`

## 3. 制订适配计划

根据差异结果，分析工作量，并制订适配计划。

### 3.1 模块关联到SaaS负责人

1. 对每个需要适配的模块，确定SaaS这边的责任人

### 3.2 安排迭代计划
	
1. 按照接口适配工作量排定迭代计划

## 4. 基于SaaS的nocalhost环境进行测试

1. 使用Java或者Kotlin开发适配器服务
2. 适配器可能根据实际情况来做一些DirtyWork（比如转发SaaS 仓库Webhook数据）
3. SaaS采用nocalhost环境作为开发环境，因此务必联系saas同学开通nocalhost

### 4.1 联系张阳开通nocalhost权限

获取登陆nocal环境的相关信息：

1. 登陆网站
2. 用户名                            
3. 密码

### 4.2 熟悉nocalhost使用

查看帮助网站：https://nocalhost.dev/zh/getting-started/

### 4.3 使用端口转发进行本地测试

如果需要连接nocalhost环境中的服务进行测试，那么需要进行端口转发。

1. 在VsCode插件中登陆Nocalhost
2. 点击`Workloads`/`Deployments`/某个服务
3. 选择`Port Forward`
4. 输入端口号`本机端口号:Nocalhost环境端口号`
5. 成功后开始联调

## 5. 联调阶段

### 5.1 镜像push

1. 在Coding上配置流水线，构建镜像并push到coding的镜像仓库
2. push的仓库地址，用户名和密码可以联系张阳

### 5.2 确定联调基础设施信息

确定联调的底层基础设施：
1. 联调staging环境的saas服务地址和端口
2. 联调staging环境的mysql数据库信息
3. 联调staging环境的redis服务器信息
4. 联调staging环境的RabbitMQ服务器地址

### 5.3 编写对应的k8s部署文件

1. 编写`apiRoute`，设置反向代理转发规则（可能需要前端调整，防止跟Saas冲突），举例：
	```
	   [
        {
          "path": "/qci/api/user/**",
          "rewritePaths": {
            "/qci/(?<segment>.*)": "/${segment}"
          },
          "routeUri": "http://e-qci-ci-saas-adapter.e-qci:8080"
        }
      ]
	```
1. 配置对应环境变量
2. 配置存活探针和有效性探针
3. 配置对应Service

### 5.3 staging环境更新

> CI之前调用基座的地址全部替换为适配器地址（GRPC/HTTP）。微服务比较多，仔细检查，减少修改遗漏

1. 申请或者找人共用devcloud机器（本地机器连不上staging环境的k8s，原因未知）
1. 联系张阳开通[部署文件仓库更新权限](git@e.coding.net:codingcorp/coding-staging-k8s.git)
2. 更新分支到`feature/qci_saas`
3. 修改适配器信息
4. push到部署文件仓库
5. 在devcloud机器上配置好k8s访问信息
6. 更新部署文件仓库
7. apply新的配置文件，升级部署

### 5.4 联调/修改BUG

1. 日志等级请设置为debug等级，方便定位