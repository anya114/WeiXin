#WeiXin#

微信公众号接口。

参考：[[接口文档]](http://mp.weixin.qq.com/wiki/home/index.html "接口文档") [[测试地址]](http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login "测试地址") [[本机公网发布]](https://ngrok.com "ngrok")

## 起步 ##

在 web.xml 中配置如下信息：

    <servlet>
        <servlet-name>weixin</servlet-name>
        <servlet-class>com.chn.wx.WeiXinServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>weixin</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>

同时在 classpath 下的 weixin.properties 里配置各项参数：

    weixin.app.id=
	weixin.app.name=
	weixin.app.secret=
	weixin.app.aeskey=
	weixin.app.token=
	
	weixin.service.package=
	
	weixin.service.async=true
	weixin.service.innerexec.size=20

解释下参数 weixin.service.async，值为 true 时异步执行，会忽略 Service 返回的任何内容，如有下行需调用客服接口，可以解决压力过大或者网络过慢等导致的“微信号暂时不能提供服务”的问题。

> **关于异步执行:** 当 weixin.service.async 值为 true 时异步执行，会忽略 Service 返回的任何内容，如有下行需调用客服接口，可以解决压力过大或者网络过慢等导致的“微信号暂时不能提供服务”的问题。
    
## 主流程 ##

`com.chn.wx.listener` 中的所有实现 `Service` 接口的类会被组装成一棵流程树，目前结点如下：

             GET
    Servlet ─────> CertifyService
       |    POST                  RAW                                 event                         CLICK
       └─────────> EncryptRouter ─────> RawMessageRouter ────────────────────────────> EventRouter ─────────> ClickEventService
                         | AES            ↑         |  text                                 |       LOCATION
                         └────> AesMessageRouter    ├───────────> TextMessageService        ├───────────────> LocationEventService
                                                    |  image                                |       SCAN
                                                    ├───────────> ImageMessageService       ├───────────────> ScanQrCodeEventService
                                                    |  video                                |       VIEW
                                                    ├───────────> VideoMessageService       ├───────────────> ViewEventService
                                                    |  voice                                |       subscribe
                                                    ├───────────> VoiceMessageService       ├───────────────> SubscribeEventService
                                                    |  location                             |       unsubscribe
                                                    ├───────────> LocationMessageService    └───────────────> UnSubscribeEventService
                                                    |  link
                                                    └───────────> LinkMessageService

结点与父结点的关系通过 `@Node(value = "raw", parent = EncryptRouter.class)` 指定。

## 消息返回 ##

`Service` 的实现方法中返回的字符串会被写回到请求流中，需要返回消息时，调用 `com.chn.wx.template.PassiveMessage` 中的对应方法生成报文返回即可。
> **注意：**仅同步执行时该返回有效

## 主动调用 ##

- `com.chn.wx.invocation.FileManager` 上传下载多媒体文件
- `com.chn.wx.invocation.GroupManager` 分组管理
- `com.chn.wx.invocation.ServiceMessageSender` 客服消息发送
- `com.chn.wx.invocation.TokenAccessor` 唯一的 token 获取入口, token 只能能过该类获取，不能另做缓存

## 语法糖 ##

`Service` 实现类中被`@Param`注解标记的字段，会被注入成 `Context` 中对应的属性值，当然也可以直接通过 `Context` 读取。
