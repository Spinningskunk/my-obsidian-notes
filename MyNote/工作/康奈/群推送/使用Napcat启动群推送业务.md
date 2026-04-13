### 1、环境配置：
jdk所需版本：17
windows版本：≥10(也有linux+docker部署的方式)
NapCat 版本：4.17.53
QQ 版本：9.9.26-44498(不必手动安装QQ napcat主程序脚本会自动安装适合当前napcat版本的QQ对应版本)

部署完成后文件目录结构将如图所示，其中：
NapCat.Shell.Windows.OneKey：Napcat脚本相关；
config.properties：Jar程序推送配置相关
konne-qq-push-1.2.2.jar：后端服务Jar包
![[Pasted image 20260403114418.png]]
### 2、启动业务：

当Jar程序启动后，在此目录下使用napcat.quick.bat 即可快速启动业务：
![[Pasted image 20260330105511.png]]

但是cmd显示的二维码貌似存在一些问题无法直接扫码，建议直接去箭头下的目录扫码即可：
![[Pasted image 20260330105752.png]]
![[Pasted image 20260330105842.png|619]]
在Napcat的命令窗口还可以看到Napcat自带的webui界面，可以查看登录账户、qq版本等各类信息
![[Pasted image 20260330110253.png]]
在Napcat启动好后，他会自动去连接我们jar程序的websocket链接，从而让jar程序可以调用收发消息的api接口；如果jar程序停止，他也会自动定时重连(30s一次),当后续jar程序再启动好后，也会自动连接成功，很方便；
![[Pasted image 20260330110416.png]]
### 3、停止业务：
Napcat的命令窗口会一直输出此qq接收和发送的所有群聊、单聊消息；
按下ctrl+c  最后停止批命令即可，或者直接关掉当前cmd命令窗口
![[Pasted image 20260330105353.png]]
### 4、Jar程序配置
后续会参考企微推送的配置文件，从而可以通过修改文件中的配置来决定消费的队列、暴露给Napcat的websocket链接、消费的MQ信息、数据消费的重试次数、随机模拟延迟的消息推送的时间配置等；
```
# RabbitMQ配置
spring.rabbitmq.host=10.1.224.93
spring.rabbitmq.port=15690
spring.rabbitmq.username=konne
spring.rabbitmq.password=konne20230312@.+

# RabbitMQ队列配置
rabbitmq.queue=konne_group_qq_push_test

# PostgreSQL配置
spring.datasource.url=jdbc:postgresql://10.1.224.93:15432/push_center_new?useUnicode=true&characterEncoding=utf8&useSSL=true&autoReconnect=true&reWriteBatchedInserts=true
spring.datasource.username=konne
spring.datasource.password=Psq@Kn2421..
spring.datasource.driver-class-name=org.postgresql.Driver

# 延迟发送配置
qq.push.delay.enabled=true
qq.push.delay.min=1500
qq.push.delay.max=6000

# 重试配置
qq.push.retry.max-count=10
qq.push.retry.delay-ms=5000

# 服务端口（可选）
server.port=8080

```