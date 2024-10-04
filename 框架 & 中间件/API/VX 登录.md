# VX 登录

[TOC]

## 准备工作

我们要先做以下准备工作：

1. 一个内网穿透的工具：https://www.ngrok.cc/user，方便进行 OAuth2.0 测试。

2. [微信公众平台接口测试帐号申请](https://link.segmentfault.com/?enc=KfLGwmSXbX0o7lcictcG%2Bw%3D%3D.Jg5AN%2BsskfNusscnU0PT6o210Fn18P6Lr4k0h%2FJxjxQgBF7Fxo7EXjjnYmq8bAFTKHlQ23cFBYP0%2F1JbRep5yg%3D%3D)

   ![{38A96F73-8701-461B-A25F-AC72F17B4F5F}](./assets/%7B38A96F73-8701-461B-A25F-AC72F17B4F5F%7D.png)

   - URL：开发者的回调地址。
   - Token：任意填写

   开发者提交信息后，**微信服务器将发送 GET 请求**到服务器地址 URL 上，其中包含一个随机字符串 echostr。我们必须做出响应这个字符串 echostr，否则配置失败：

   ~~~java
   @RequestMapping("wechat")
   @RestController
   @Slf4j
   public class WechatController {
       private final WeChatMpService weChatMpService;
   
       public WechatController(WeChatMpService weChatMpService) {
           this.weChatMpService = weChatMpService;
       }
   
       @GetMapping
       public void get(@RequestParam(required = false) String signature,
                       @RequestParam(required = false) String timestamp,
                       @RequestParam(required = false) String nonce,
                       @RequestParam(required = false) String echostr,
                       HttpServletResponse response) throws IOException {
           // 这里的 WeChatMpService 是我们自定义的，继承 weixin-java-mp 包的 WxMpService 类，来做认证服务。下面会给出代码
           if (!this.weChatMpService.checkSignature(timestamp, nonce, signature)) {
               log.warn("接收到了未通过校验的微信消息");
               return;
           }
           response.setCharacterEncoding("UTF-8");
           response.getWriter().write(echostr);
           response.getWriter().flush();
           response.getWriter().close();
           log.info("配置成功");
       }
   }
   ~~~

   

   

导入 Maven 依赖：

~~~xml
<dependency>
    <groupId>com.github.binarywang</groupId>
    <artifactId>weixin-java-mp</artifactId>
    <version>4.4.0</version>
</dependency>
~~~

配置编写：

~~~yaml
wx:
  mp:
    appid: wxaf7fe05a8xxxxxxxxx
    secret: 57b48fcec2d5db1axxxxxxxxxxx
    token: yunzhi
    aesKey: XXXXXX # 消息的加密密钥，无需设置
~~~

~~~java
@ConfigurationProperties("wx.mp")
@Configuration
@Data
public class WxMpConfig {
    private String appId;
    private String secret;
    private String token;
    private String aesKey;
}
~~~

初始化 WxMpService

~~~java
@Service
public class WeChatMpService extends WxMpServiceImpl {
    private final WxMpConfig wxMpConfig;

    public WeChatMpService(WxMpConfig config) {
        this.wxMpConfig = config;
    }

    @PostConstruct
    public void init() {
        final WxMpDefaultConfigImpl config = new WxMpDefaultConfigImpl();
        config.setAppId(wxMpConfig.getAppId());
        config.setSecret(wxMpConfig.getSecret());
        config.setToken(wxMpConfig.getToken());
        super.setWxMpConfigStorage(config);
    }
}
~~~



access_token 是公众号的全局唯一接口调用凭据，开发者需要进行妥善保存。有效期目前为 2 个小时，需定时刷新，重复获取将导致上次获取的 access_token 失效。weixin-java-mp 包会帮我们做这件事，因此我们无需操心这一步。



同一用户，对同一个微信开放平台帐号下的不同应用，UnionID 是相同的，但 openID 是不同的。



## 展示二维码

接着我们要生成二维码（[微信获取二维码文档](https://link.segmentfault.com/?enc=zv28EVNQlYC0lxEVhG95gw%3D%3D.pDmfOV38XZOX7wiETYbVbjwAZkShzd0phC9UFidD21LACnYz1oy2BA9Jn3ODDyvqXHjWwJbEy07j4cB3kkk1D39nebx6hgbgNjuCQ6VydAt5bQSAon1fI%2Fid0h8r9YwZ7%2F%2BoRMnMakjigPfh9Pg%2BrQ%3D%3D)）。目前有 2 种类型的二维码：

- 临时二维码，有过期时间的，最长可以设置 30 天，能够生成较多数量。
- 永久二维码，无过期时间的，数量较少（目前为最多10万个）。



| 参数           | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| expire_seconds | 该二维码有效时间，以秒为单位。 最大不超过2592000（即30天），此字段如果不填，则默认有效期为60秒。 |
| action_name    | 二维码类型，QR_SCENE为临时的整型参数值，QR_STR_SCENE为临时的字符串参数值，QR_LIMIT_SCENE为永久的整型参数值，QR_LIMIT_STR_SCENE为永久的字符串参数值 |
| action_info    | 二维码详细信息                                               |
| scene_id       | 场景值ID，临时二维码时为 32 位非 0 整型，永久二维码时最大值为100000（目前参数只支持1--100000） |
| scene_str      | 场景值ID（字符串形式的ID），字符串类型，长度限制为1到64      |

生成步骤为：

1. 首先要获取二维码的 ticket，ticket 需要提供一个开发者自行设定的参数（scene_id），用来标识一个请求上下文。

   ~~~java
   String qrUuid = UUID.randomUUID().toString();
   WxMpQrCodeTicket wxMpQrCodeTicket = this.wxMpService.getQrcodeService().qrCodeCreateTmpTicket(qrUuid, 10 * 60);
   ~~~

   临时二维码的 ticket 请求：

   ~~~json
   HTTP POST https://api.weixin.qq.com/cgi-bin/qrcode/create?access_token=TOKEN
   {"expire_seconds": 604800, "action_name": "QR_SCENE", "action_info": {"scene": {"scene_id": 123}}}
   ~~~

   返回的结果：

   ```json
   {"ticket":"gQH47joAAAAAAAAAASxodHRwOi8vd2VpeGluLnFxLmNvbS9xL2taZ2Z3TVRtNzJXV1Brb3ZhYmJJAAIEZ23sUwMEmm
   3sUw==","expire_seconds":60,"url":"http://weixin.qq.com/q/kZgfwMTm72WWPkovabbI"}
   ```

2. 凭借 ticket 到指定 URL 换取二维码。

   ~~~java
   wxMpQrCodeTicket.getToken();
   ~~~

   ~~~json
   HTTP GET https://mp.weixin.qq.com/cgi-bin/showqrcode?ticket=gQGn7zwAAAAAAAAAAS5odHRwOi8vd2VpeGluLnFxLmNvbS9xLzAydVJhT3RoUE1jWEcxVC1rWk5DY3AAAgSOPf1mAwRwFwAA
   ~~~

   在 ticket 正确情况下，http 返回码是200，是一张图片，可以直接展示或者下载。

## 事件推送

微信用户使用公共号时可能会产生很多事件，例如

- 关注/取消关注事件
- 扫描带参数二维码事件
- 自定义菜单事件
- 点击菜单拉取消息时的事件推送
- 点击菜单跳转链接时的事件推送

[微信推送事件文档](https://link.segmentfault.com/?enc=FnB9V7EQqSlqES7Sy8QXCg%3D%3D.Mztg3yGhOtf2K9QRQp1VzOjfEW5nvN4lgSOTFm5TvIdx2vNzUAJSVoVIBbZxRTFX1zjeZLTa2qtG%2FCxGQ2w22RHCvHd1JbVYSXc2mthYJ03M4C1ZErHYk5ax%2BwxPhmAB)

用户扫描带场景值二维码时，可能推送以下两种事件：

- 如果用户还未关注公众号，则用户可以关注公众号。关注后，微信会将带场景值关注事件推送给开发者。
- 如果用户已经关注公众号，则微信会将带场景值扫描事件推送给开发者。

~~~java
@RestController
public class WechatController {
    @PostMapping(produces = "text/xml; charset=UTF-8")
    public void api(HttpServletRequest httpServletRequest,
                    HttpServletResponse httpServletResponse,
                    @RequestParam("signature") String signature,
                    @RequestParam(name = "encrypt_type", required = false) String encType,
                    @RequestParam(name = "msg_signature", required = false) String msgSignature,
                    @RequestParam("timestamp") String timestamp,
                    @RequestParam("nonce") String nonce) throws IOException {

        if (!this.weChatMpService.checkSignature(timestamp, nonce, signature)) {
            log.warn("接收到了未通过校验的微信消息");
            return;
        }

        // 构建 Body
        BufferedReader bufferedReader = httpServletRequest.getReader();
        StringBuilder requestBodyBuilder = new StringBuilder();
        String str;
        while ((str = bufferedReader.readLine()) != null) {
            requestBodyBuilder.append(str);
        }
        String requestBody = requestBodyBuilder.toString();

        log.info("\n接收微信请求：[signature=[{}], encType=[{}], msgSignature=[{}],"
                 + " timestamp=[{}], nonce=[{}], requestBody=[\\n{}\\n]",
                 signature, encType, msgSignature, timestamp, nonce, requestBody);
        log.info(httpServletRequest.getQueryString());
        log.info(httpServletRequest.getContentType());



        String out = null;
        if (encType == null) {
            // 明文传输的消息
            WxMpXmlMessage inMessage = WxMpXmlMessage.fromXml(requestBody);
            log.info("事件为 " + inMessage.getEventKey());

            // 从这里进行路由分发
            WxMpXmlOutMessage outMessage = this.weChatMpService.route(inMessage);
            if (outMessage == null) {
                httpServletResponse.getOutputStream().write(new byte[0]);
                httpServletResponse.flushBuffer();
                httpServletResponse.getOutputStream().close();
                return;
            }
            out = outMessage.toXml();
        } else if ("aes".equals(encType)) {
            // aes 加密的消息
            WxMpXmlMessage inMessage = WxMpXmlMessage.fromEncryptedXml(requestBody,
                                                                       this.weChatMpService.getWxMpConfigStorage(), timestamp, nonce, msgSignature);
            WxMpXmlOutMessage outMessage = this.weChatMpService.route(inMessage);
            if (outMessage == null) {
                httpServletResponse.getOutputStream().write(new byte[0]);
                httpServletResponse.flushBuffer();
                httpServletResponse.getOutputStream().close();
                return;
            }
            out = outMessage.toEncryptedXml(this.weChatMpService.getWxMpConfigStorage());
        }

        httpServletResponse.getOutputStream().write(out.getBytes(StandardCharsets.UTF_8));
        httpServletResponse.flushBuffer();
        httpServletResponse.getOutputStream().close();
    }
}
~~~



这里我们引入 WxMpMessageRouter 路由器，来做事件分发的工作。

~~~java
@Service
public class WeChatMpService extends WxMpServiceImpl {
    private WxMpMessageRouter router;

    private void refreshRouter() {
        final WxMpMessageRouter newRouter = new WxMpMessageRouter(this);

        // 关注事件
        newRouter.rule().async(false)
            .msgType(EVENT).event(SUBSCRIBE)
            .handler(this.subscribeHandler).end();

        // 取消关注事件
        newRouter.rule().async(false)
            .msgType(EVENT).event(UNSUBSCRIBE)
            .handler(this.unsubscribeHandler).end();

        // 扫码事件 
        newRouter.rule().async(false)
            .msgType(EVENT).event(SCAN)
            .handler(this.scanHandler).end();

        // 默认
        newRouter.rule().async(false)
            .handler(this.msgHandler).end();
        this.router = newRouter;
    }

   /**
   * 微信事件通过这个入口进来
   * 根据不同事件，调用不同handler
   */
    public WxMpXmlOutMessage route(WxMpXmlMessage message) {
        try {
            return this.router.route(message);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }

        return null;
    }
}
~~~

**关注/取消关注事件：**

|     参数     | 描述                                             |
| :----------: | ------------------------------------------------ |
|  ToUserName  | 开发者微信号                                     |
| FromUserName | 发送方账号（一个OpenID）                         |
|  CreateTime  | 消息创建时间 （整型）                            |
|   MsgType    | 消息类型，event                                  |
|    Event     | 事件类型，subscribe(订阅)、unsubscribe(取消订阅) |

扫描带参数二维码事件，**用户未关注时，进行关注后的事件推送**

|     参数     | 描述                                               |
| :----------: | :------------------------------------------------- |
|  ToUserName  | 开发者微信号                                       |
| FromUserName | 发送方账号（一个OpenID）                           |
|  CreateTime  | 消息创建时间 （整型）                              |
|   MsgType    | 消息类型，event                                    |
|    Event     | 事件类型，subscribe                                |
|   EventKey   | 事件 KEY 值，qrscene_ 为前缀，后面为二维码的参数值 |
|    Ticket    | 二维码的 ticket，可用来换取二维码图片              |

扫描带参数二维码事件，**用户已关注时的事件推送**：

| 参数         | 描述                                  |
| :----------- | :------------------------------------ |
| ToUserName   | 开发者微信号                          |
| FromUserName | 发送方账号（一个OpenID）              |
| CreateTime   | 消息创建时间 （整型）                 |
| MsgType      | 消息类型，event                       |
| Event        | 事件类型，SCAN                        |
| EventKey     | 事件 KEY 值，没有 qrscene_ 作为前缀   |
| Ticket       | 二维码的 ticket，可用来换取二维码图片 |



下面给出部分 Handler 的代码

~~~java
@Component
public class SubscribeHandler implements WxMpMessageHandler {
    @Override
    public WxMpXmlOutMessage handle(
            WxMpXmlMessage wxMessage,
            Map<String, Object> context,
            WxMpService wxMpService, WxSessionManager sessionManager) throws WxErrorException {
        String key = wxMessage.getEventKey();
        if (key.isEmpty() || key.isBlank()) {
            // 直接关注的，并不是扫码过来的
        }
        if (key.startsWith("qrscene_")) {
            // 首先未关注扫码，然后关注后的事件
            key = key.substring(8);
        } else {
            // 关注后扫码的
        }
        
        return WxMpXmlOutMessage.TEXT().content("感谢关注，祝您生活愉快!")
                .fromUser(wxMessage.getToUser()).toUser(wxMessage.getFromUser())
                .build();
    }
}
~~~
