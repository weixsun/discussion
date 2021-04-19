自Chrome 51版本开始，浏览器的 Cookies 新增了一个`SameSite`属性，用来防止 CSRF 攻击和信息泄漏，更多信息参考chrome [Feature: 'SameSite' cookie attribute](https://www.chromestatus.com/features/4672634709082112)。

![](https://gitee.com/weixsun/imgs/raw/master/20210415103959.png)

### 简单回顾什么是CSRF攻击

`Cookies`往往用来存储用户的身份信息，恶意网站通过设法伪造带有正确`Cookies`进行 HTTP 请求，这就是 CSRF 攻击。

举例来说，用户登陆了银行网站`your-bank.com`，银行服务器发来了一个 Cookie。

```text
Set-Cookie: session_id=abc123;
```

用户后来又访问了恶意网站`malicious-site.com`，恶意网站总是想方设法让你在恶意站点发送一个表单请求。

> 手段：中奖填写联系信息、透明的form表单提交按钮、附加在诱惑图片上的超链接

```html
<form action="your-bank.com/transfer" method="POST">
  ...
</form>
```

用户一旦被诱骗发送这个表单，银行网站就会收到带有正确 Cookie 的请求。为了防止CSRF攻击，银行网站表单会设置一个隐藏域，表单提交时一起带上一个随机token至服务器，告诉服务器这是真实请求。

```html
<form action="your-bank.com/transfer" method="POST">
  <input type="hidden" name="token" value="dad3weg34">
  ...
</form>
```

之所以被CSRF攻击，是因为恶意网站诱导你在其页面上发送了第三方cookie(此时的银行网站cookie为第三方cookie)，它除了用于 CSRF 攻击，还可以用于用户追踪。

> 第三方是一种相对概念，规定假定你正在访问的站点为第一方站点，则浏览器为第二方，其他网站就是第三方站点，第三方站点诱导你发送第一方站点的cookie时，我们就说此时的cookie为第三方cookie。

比如，Facebook 在第三方网站插入一张看不见的图片。

```html
<img src="facebook.com" style="visibility:hidden;">
```

浏览器加载上面代码时，就会向 Facebook 发出带有 Cookie 的请求，从而 Facebook 就会知道你是谁，访问了什么网站。

### SameSite

cookies机制一直被认为是不安全的，随着技术的更新，界内一直在完善cookies的安全机制，`SameSite`属性是谷歌浏览器为完善cookies安全机制的出的特性之一。


Cookie 的`SameSite`属性用来限制第三方 Cookie的行为。

它可以设置三个值。

- Strict
- Lax
- None

#### Strict

`Strict`最为严格，完全禁止第三方 Cookie，当当前站点与请求目标站点是`跨站`关系时，总是不会发送 Cookie。换言之，只有当前站点 与请求目标站点是`同站`关系时，才会带上 Cookie。

```text
Set-Cookie: CookieName=CookieValue; SameSite=Strict;
```

这个规则过于严格，可能造成非常不好的用户体验。

举例说明：

假定你当前所处站点地址为`https://obmq.com/index.html`，该站需要通过XHR请求获取某天气站点的未来7天的天气信息`https://weather-forecast.org/api/weather?future=7`，该接口要求必须携带cookie`Set-Cookie: vip=true; Path=/; HttpOnly; SameSite=Strict;`，这种情况下，你无论如何都无法在`https://obmq.com`站点下发送这个XHR请求时还能携带上这个cookie，换句话说，你发送的接口请求的cookie请求头一定不会有`vip=true`，即使你现在已经是该天气网站的vip。

#### None

`None`在Chrome 85 版本之前是`SameSite`的默认设置值，即`Set-Cookie: key=value; SameSite=None`等于`Set-Cookie: key=value`。

在Chrome 85 版本之前，显示设置`SameSite=None`不需要设置`Secure`属性，详细参见:[Reject insecure SameSite=None cookies](https://www.chromestatus.com/feature/5633521622188032)

在Chrome 85 版本以后，站点选择显式关闭`SameSite`属性时，在将其值设为`None`的同时。必须同时设置`Secure`属性（表示Cookie 只能通过 HTTPS 协议发送），否则无效。

下面的设置有效。

```text
Set-Cookie: widget_session=abc123; SameSite=None; Secure
```

![](https://gitee.com/weixsun/imgs/raw/master/20210415130742.png)

下面的设置无效。

```text
Set-Cookie: widget_session=abc123; SameSite=None
```

![](https://gitee.com/weixsun/imgs/raw/master/20210415130541.png)


#### Lax

Chrome 在85版本后将`Lax`设为`SameSite`的默认值，即`Set-Cookie: key=value; SameSite=Lax`等于`Set-Cookie: key=value`，详细参见:[Cookies default to SameSite=Lax](https://www.chromestatus.com/features/5088147346030592)。

`Lax`规则比较宽松，大多数情况也不发送第三方 Cookie，但是导航到目标站点的 Get 请求除外。

```text
Set-Cookie: CookieName=CookieValue; SameSite=Lax;
```

导航到目标站点的 GET 请求，只包括三种情况：链接，预加载请求，GET 表单。详见下表。

请求类型 | 示例 | SameSite=None;Secure | Lax
---|---|---|---
链接 | &lt;a href="..."&gt;&lt;/a&gt; | 发送 Cookie | 发送 Cookie
预加载 | &lt;link rel="prerender" href="..."/&gt; | 发送 Cookie | 发送 Cookie
GET 表单 | &lt;form method="GET" action="..."&gt; | 发送 Cookie | 发送 Cookie
POST 表单 | &lt;form method="POST" action="..."&gt; | 发送 Cookie | 不发送
iframe | &lt;iframe src="..."&gt;&lt;/iframe&gt; | 发送 Cookie | 不发送
xhr/fetch | $.get("...") | 发送 Cookie | 不发送
Image | &lt;img src="..."&gt; | 发送 Cookie | 不发送
Script | &lt;script src="..."&gt; | 发送Cookie | 不发送


设置了`Strict`或`Lax`以后，基本就杜绝了 CSRF 攻击。当然，前提是用户浏览器支持 `SameSite` 属性。

### Schemeful Same-Site

自Chrome 86版本开始，考虑到不安全的`http://`协议仍然为网络攻击者提供了篡改cookie的机会，然后将这些cookie用于站点安全的`https://`。谷歌浏览器修改了cookie的`Same Site`的定义，将在相同域名的安全（https://）协议和不安全（http://）协议作为是否`跨站`的判断因素之一。详情参见[Feature: Schemeful same-site](https://www.chromestatus.com/feature/5096179480133632)

如果您的站点全面升级到https协议，那么下面的内容不适合您；如果您的站点是https协议和http协议混存，那么您需要关注。

#### 常见的 "cross-scheme" Cookies携带情况

##### 超链接

当`Schemeful Same-Site`禁止时，从**http://** site.example`链接到`**https**://site.example时，即使`SameSite=Strict`依然会携带上cookie。

当`Schemeful Same-Site`启用时，从**http://** site.example`链接到`**https**://site.example时，`SameSite=Strict`的cookies会被锁定，其他cookies携带的表现如下图和下表：

![](https://gitee.com/weixsun/imgs/raw/master/20210415175419.png)

<figcaption style="text-align: center;">Cross-scheme navigation from HTTP to HTTPS.</figcaption>

<br/>

<table>
  <tr>
   <td>
   </td>
   <td><strong>HTTP → HTTPS</strong>
   </td>
   <td><strong>HTTPS → HTTP</strong>
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Strict</code>
   </td>
   <td>⛔ Blocked
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Lax</code>
   </td>
   <td>✓ Allowed
   </td>
   <td>✓ Allowed
   </td>
  </tr>
  <tr>
   <td><code>SameSite=None;Secure</code>
   </td>
   <td>✓ Allowed
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
</table>

##### 加载子资源

加载子资源的方式包括`images`, `iframes`, 和`XHR or Fetch`的网络请求。

加载子资源分为`http加载https子资源`和`https加载子http资源`，携带cookies的表现如下图和下表：

当`Schemeful Same-Site`禁止时，加载子资源时，`SameSite=Strict` 或者`SameSite=Lax`的cookie会被携带上。

当`Schemeful Same-Site`启用时，加载子资源时，`SameSite=Strict`或者`SameSite=Lax`的cookies会被锁定，其他cookies携带的表现如下图和下表：

![](https://gitee.com/weixsun/imgs/raw/master/20210415181427.png)

<figcaption style="text-align: center;">An HTTP page including a cross-scheme subresource via HTTPS.</figcaption>

<br/>

<table>
  <tr>
   <td>
   </td>
   <td><strong>HTTP → HTTPS</strong>
   </td>
   <td><strong>HTTPS → HTTP</strong>
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Strict</code>
   </td>
   <td>⛔ Blocked
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Lax</code>
   </td>
   <td>⛔ Blocked
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
  <tr>
   <td><code>SameSite=None;Secure</code>
   </td>
   <td>✓ Allowed
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
</table>

##### POST 表单


当`Schemeful Same-Site`禁止时，发送POST表单时，`SameSite=Strict` 或者`SameSite=Lax`的cookie会被携带上。

当`Schemeful Same-Site`启用时，发送POST表单时，只有`SameSite=None`的cookies会被携带，其他cookies携带的表现如下图和下表：

![](https://gitee.com/weixsun/imgs/raw/master/20210415181704.png)

<figcaption style="text-align: center;">Cross-scheme form submission from HTTP to HTTPS.</figcaption>

<br/>

<table>
  <tr>
   <td>
   </td>
   <td><strong>HTTP → HTTPS</strong>
   </td>
   <td><strong>HTTPS → HTTP</strong>
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Strict</code>
   </td>
   <td>⛔ Blocked
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
  <tr>
   <td><code>SameSite=Lax</code>
   </td>
   <td>⛔ Blocked
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
  <tr>
   <td><code>SameSite=None;Secure</code>
   </td>
   <td>✓ Allowed
   </td>
   <td>⛔ Blocked
   </td>
  </tr>
</table>


##### 对WebSockets的影响？

如果WebSocket连接与页面的安全性相同，则仍将被视为同站。

`https://`连接`wss://`被视为同站；`http://`连接`ws://`被视为同站，否则视为跨站。详细如下：

Same-site:

- `wss://` 从 `https://` 连接
- `ws://` 从 `http://` 连接

Cross-site:

- `wss://` 从 `http://` 连接
- `ws://` 从 `https://` 连接


### 最佳实践

#### 依然使用token机制防止CSRF攻击

设置了SameSite属性值为`Strict`或`Lax`以后，基本杜绝了 CSRF 攻击。但是SameSite是Cookies属性之一，所以存在以下诸多限制：

- 要求浏览器必须兼容Cookies的SameSite属性

- 要求客户端应用必须支持Cookies机制，比如APP和微信小程序并不支持Cookies

> 限制如上所列，但不限于所列。

因为SameSite属性存在以上限制，所以需要服务器端依然采用token机制来防止CSRF攻击。


#### 显示指定SameSite属性值为`Lax`或`Strict`

总应该设置一个显式的`SameSite`属性，而不是依赖浏览器为您应用的默认设置。这使您对Cookie的使用意图更加明确，并提高了跨浏览器获得一致体验的机会。

#### 不兼容的客户端的处理

对于`SameSite`属性的兼容性在不同浏览器和相同浏览器的不同版本之间是不同的，可以参考chromium.org上的更新页面的[已知的不兼容客户端](https://www.chromium.org/updates/same-site/incompatible-clients)以了解当前已知的问题，但是无法确定是否详尽无遗。 尽管这不是理想的选择，但可以在此过渡阶段中采用一些解决方法。

**方式一：同时设置兼容SameSite属性的客户端和不兼容SameSite属性的客户端**

```text
// ~ 同时设置兼容SameSite属性的客户端和不兼容SameSite属性的客户端

// ~ 对于兼容SameSite属性的客户端，显示指定SameSite属性值以明确使用意图
Set-cookie: 3pcookie=value; SameSite=None; Secure

// ~ 对于不兼容SameSite属性的客户端，不设置SameSite属性
Set-cookie: 3pcookie-legacy=value; Secure
```

下面的示例显示了如何使用[Express框架](https://expressjs.com/)及其[cookie-parser](https://www.npmjs.com/package/cookie-parser)中间件在Node.js中执行此操作。

```javascript
const express = require('express');
const cp = require('cookie-parser');
const app = express();
app.use(cp());

app.get('/set', (req, res) => {
  // Set the new style cookie
  res.cookie('3pcookie', 'value', { sameSite: 'none', secure: true });
  // And set the same value in the legacy cookie
  res.cookie('3pcookie-legacy', 'value', { secure: true });
  res.end();
});

app.get('/', (req, res) => {
  let cookieVal = null;

  if (req.cookies['3pcookie']) {
    // check the new style cookie first
    cookieVal = req.cookies['3pcookie'];
  } else if (req.cookies['3pcookie-legacy']) {
    // otherwise fall back to the legacy cookie
    cookieVal = req.cookies['3pcookie-legacy'];
  }

  res.end();
});

app.listen(process.env.PORT);
```

**方式二：判断客户端的`user-agent`**

在发送`Set-Cookie`相应头时，您可以选择通过`user-agent`字符串检测客户端。请参阅[不兼容客户端的列表](https://www.chromium.org/updates/same-site/incompatible-clients)，建议您找一个工具库来处理`user-agent`，因为您很可能不想自己编写这些正则表达式。

这种方法的好处在于，它只需要在设置cookie时进行一次更改即可。 但是，此处必须指出是`user-agent`嗅探本身是不可靠的，并且可能无法捕获所有受影响的用户。

无论选择哪种方式，建议确保有一种记录低版本客户端比例的方法。 一旦比例下降到网站可接受的阈值以下，请确保您有提醒或警报以删除此替代方法。

#### 谷歌计划推出"Privacy Sandbox"

谷歌计划完全禁止第三方cookie，毕竟cookie真的不是很安全，详细参考[this](https://blog.chromium.org/2020/10/progress-on-privacy-sandbox-and.html)



### 参考链接

- [SameSite cookies explained](https://web.dev/samesite-cookies-explained)

- [SameSite cookies recipes](https://web.dev/samesite-cookie-recipes/)

- [Schemeful Same-Site](https://web.dev/schemeful-samesite)

- [Cookie 的 SameSite 属性 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html)

---

文章同步发布在各大主流知识共享平台，所以设`github`为统一的反馈区

疑问、讨论、问题反馈：https://github.com/weixsun/discussion

---

关注微信公众号 **obmq** 及时了解最新动态

![](https://gitee.com/weixsun/imgs/raw/master/obmq-mp-weixin.png)