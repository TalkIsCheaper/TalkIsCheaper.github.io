# 		CAS单点登录(四)----自定义认证登录策略

#### 前提：

​			加入CAS框架提供的方案还是不能满足我们的需求，比如我们不紧需要用户名和密码，还要验证其他信息，比如邮箱、手机号、但是邮箱、手机号信息在另外一个数据库，还有一段时间内同一个IP输入错误次数的限制等等。我们就需要自定义认证策略，自定义CAS的web认证流程。

### 自定义认证校验策略

​			CAS提供了多种认证数据源，例如JDBC，File，JSON等，但是如果我们想在自己的认证方式中提供不同的数据源选择，就需要我们自己去实现自定义认证。

​			自定义策略的主要通过更改CAS配置，通过 AuthenticationHandler （ 身份验证处理程序 ）在CAS中设计和注册定义身份验证策略，拦截数据源达到目的。

##### 主要分为三个步骤

1. 设计自己的认证处理数据的程序。
2. 注册认证拦截器到CAS的认证引擎中
3. 更改认证配置到CAS中

