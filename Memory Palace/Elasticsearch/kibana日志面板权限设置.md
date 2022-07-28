# 文档目的

在使用kibana作为日志中心的client时，需要对用户进行权限控制。通过使用kibana的角色权限和面板配置，可以对平台用户进行索引级别，面板级别的权限控制。此文档介绍了如何创建一个仅可查看日志的kibana面板，供平台用户查看yarn日志使用。



## 1. 登录kibana

使用管理员账号登录kibana

默认管理员用户是

<span style="color:green">elastic</span>

在部署ES生成密钥时会生成相应的密码

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415110055503.png" alt="image-20220415110055503" style="zoom:67%;" />



## 2. 创建yarn日志查看空间

### 创建新空间

首次登录后通常会引导至默认页面，点击默认页面的左上角，会弹出 ` 管理空间` 按钮。

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415110842120.png" alt="image-20220415110842120" style="zoom: 67%;" />



也可以通过侧边栏 ` Stack Management -> 工作区 ` 按钮来进入工作区管理按钮。

![image-20220415111440708](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\working_area)

点击 ` 创建工作区` ，**创建一个单独用来查看yarn日志的工作区**。

![image-20220415111712180](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_new_working_area1)

填写工作区相关信息，配置用户可见的kibana面板，只保留 ` Logs` 和 ` 高级设置` 两个面板，其余全部隐藏。配置完后，点击 ` 创建工作区`

![image-20220415112254965](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_new_working_area2)



### 修改空间相关配置

需要修改空间中的



![image-20220415143204630](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_new_working_area3)



## 3. 创建角色和用户

### 创建角色

kibana用户可访问的索引时通过角色来进行控制的，所以首先创建` 角色`  。

侧边栏 `Stack Management` -> `角色`, 打开页面之后右上角点击`创建角色`。

![image-20220415113549217](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_roles1)

在创建角色的页面中，填写`角色名称`、`索引`、`权限`三项，`集群权限`和`运行身份权限`不要给。PS: 为了避免用户可查看到其他索引，建议索引权限的配置项采取 **一个角色只能查看一类日志** 的配置策略。

![image-20220415114030551](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_roles2)

下一步，点击添加kibana访问权限。

![image-20220415114303588](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Elasticsearch\pics\kibana\create_roles3)

在弹窗中，配置该角色的访问权限。选择刚刚创建的`工作区`，`**所有功能的权限**` 中选择 `Customize`，只在`Observability`中开启`Logs`的`Read`权限。

配置完成后，点击`添加Kibana权限`按钮。

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415114655681.png" alt="image-20220415114655681" style="zoom:67%;" />

全部配置完后点击`创建角色`按钮。新的角色列表中就会出现刚才创建的`角色`

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415115043028.png" alt="image-20220415115043028" style="zoom:67%;" />

### 创建用户

侧边栏 `Stack Management` -> `用户`, 打开页面之后右上角点击`创建用户`。

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415115240182.png" alt="image-20220415115240182" style="zoom:67%;" />



输入用户的

`用户名`（登陆时使用）

`全名`

`密码`（用户登陆后可自行修改）

`角色`(就是刚才的角色)



<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415115405438.png" alt="image-20220415115405438" style="zoom:67%;" />

点击`创建用户`，新的用户会显示在用户列表中

<img src="D:\liuming\AppData\Roaming\Typora\typora-user-images\image-20220415142623026.png" alt="image-20220415142623026" style="zoom:67%;" />





## 4. 创建索引模式

设置日志模板之前，需要配置一个`索引模式`











