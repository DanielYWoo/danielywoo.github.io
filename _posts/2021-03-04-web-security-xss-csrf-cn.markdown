---
layout: post
title:  "Web 安全指导 - XSS and CSRF"
subtitle:  "XSS & CSRF 安全规范，跨站伪造请求不需要CSRF Token"
date:   2021-03-04 18:00:00
published: true
---

## 概述

许多人对XSS和CSRF之间的差异没有清晰的了解，因此在选择安全策略以防止XSS和CSRF时会感到困惑，结果发现许多人只是在不理解攻击场景的情况下盲目使用所有的安全武器，这有时是不必要的，比如本文将会告诉你CSRF token不是必要的，这可能会让很多人觉得很诧异。在本文中，我将尝试帮助您了解它们之间的差异，为您提供最佳选择，并说明安全性与实现工作之间的权衡。


## XSS

### 分类

我们曾经有两种类型的XSS，即类型1和类型2。在2005年，Amit Klein发现了第三种类型的XSS，通常称为类型0。

#### 类型1：stored XSS
存储的XSS意味着可以将恶意脚本注入到服务器存储中，例如后端数据库或[浏览器的Local Storage](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API)，然后脚本可以向服务器发送请求，以窃取用户敏感信息，甚至进行付款。这种攻击通常是永久性的。

#### 类型2：reflected XSS
Reflected XSS是指应用的错误消息或搜索结果包含来自用户输入的恶意脚本。该脚本未存储在任何地方。与类型1不同，这种攻击不是永久性的，需要用户提交触发。

#### 类型0：基于DOM的XSS
类型1和类型2中的恶意输入提交给服务器处理后，攻击最有可能发生在服务器渲染的页面和表单中。但是在类型0中，恶意的用户输入被直接注入DOM（例如document.write），而没有任何服务器交互。这种类型在现代HTML5单页应用程序中更频繁地发生。

### 防止XSS攻击

接下来我们从客户端渲染，服务器端渲染，API设计等多个方面来看如何防范XSS攻击。

#### 客户端渲染 (Client Side Rendering)

大多数客户端框架具有内置的XSS预防功能。例如，React自动转义变量。例如
```
const test = '< > & <script>alert(1);</script>'
return (<div>test</div>)
```
结果是
```
<div>&lt; &gt; &amp; &lt;script&gt;alert(1);&lt;/script&gt;</div>
```

在大多数情况下React不受XSS的影响，除非您使用某些特殊的高风险功能，比如以下三种：

##### 第一种 - inline javascript:code
```
const val = "javascript:alert(0)"
return (<a href={val}>mylink</a>)
```

##### 第二种 - base64 encoded script
```
const val = "data:text/html;base64," + base64encode("<script>alert(0)</script>")
return (<a href={val}>mylink</a>)
```
##### 第三种 - dangerouslySetInnerHTML
```
{% raw %}
<div dangerouslySetInnerHTML={{__html:test}} />
{% endraw %}
```

#### 服务端渲染 (Server Side Rendering)
我不想过多地谈论SSR（服务器端渲染），因为无论JSP，PHP，FreeMarker还是Thymeleaf，这种模式都将消失。我们仍然使用SSR的唯一场景是SEO。

>尽管Google爬虫可以模拟无头浏览器并搜寻javascript加载的内容，但由于它相对较慢，因此可能会超出crawler budget，因此SEO场景下建议您预先渲染静态内容或使用SSR。

最受欢迎的渲染解决方案都是基于nodejs的，大多数受欢迎的框架（如React）都可以在服务器端进行渲染，因为React已经有一些机制（在上一节中提到过）可以防止XSS，因此无需担心额外的安全问题，并且SSR或客户端浏览器生成的页面是相同的。

#### Server API Response
与具有很少API的传统JSP / PHP / Thymeleaf应用程序不同，现代应用程序基于许多API。如果API响应为JSON格式，并且您使用的是诸如React之类的现代框架，则无需转义HTML实体（例如“ <”和“>”）。如果您使用的是jQuery之类的古老框架，则需要使用“$('id').text(response)”而不是“ $('id').html(response)”。

#### 输入清理 (Input Sanitization)
有人认为我们应该始终进行输入清理，在将数据提交到服务器时避免所有潜在的危险字符。例如，如果用户在textarea控件中输入一个数学表达式“ 2>1”，我们将数据转换为“ 2&gt;1”，并保存在数据库中。但这不是我们要存储的数据，如果您使用React以JSON格式返回响应，API将返回“2&gt;1”然后页面会被渲染为“2&amp;gt;1”，这肯定不是你想要的。只有您使用古老的SSR框架（如JSP / PHP）默认不做转义的时候才能正确呈现。甚至JSTL也会错误地显示它，如下所示

```
<c:out value="${val}"/>
2&amp;gt;1
```
如果你自以为聪明，你可能这样来解决这个问题：
```
<c:out value="${val}" escapeXml="false"/>
2&gt;1
```

但是此解决方案中，您必须确保数据库中没有存储任何特殊字符而非恶意脚本。 这样您将永远无法创建像github.com或mathway.com这样的网站。 您应该将“ 2>1”原样存储在数据库中，并且推迟到使用和渲染的时候再决定是否对其进行转义以及如何对其进行转义。

在这里，我们仅讨论XSS的输入清理，对于其他情况，输入清理非常有用，例如，防止用户输入一个值以使应用程序溢出以返回包含诸如堆栈跟踪之类的敏感信息的错误消息。

有人争辩说，如果不进行输入清理，渲染页面风险太大。我认为输入清理只能heuristically地猜测XSS，猜测很不靠谱，他无法区分“小于号”是个数学计算公式的内容还是个脚本标签的开始，它只是在使您的应用程序渲染行为不正确。您完全可以使用现代框架来解决这些问题，没必要为了用老旧的框架，而使用输入清理来牺牲程序正确性。

原则：输入清理不是用来防止XSS（或SQL注入）的，保持用户输入原样，仅在使用和渲染时决定如何对其进行转义。

#### CSP 和其它 headers

##### Content Security Policy (CSP)
CSP是一个非常强大的工具，它可以防止浏览器加载不受信任的脚本和样式表以避免XSS，并且可以指定非常复杂的策略。 例如，您可以禁止任何cross-origin资源通过以下方式加载
```
Content-Security-Policy: default-src 'self'
```

或者，也许您想仅允许从某些来源加载图像
```
Content-Security-Policy: default-src 'self'; img-src img.abc.com
```
如果黑客将脚本注入您的网页，则浏览器将不会加载该脚本。

除了防止XSS，CSP还有很多有用的特性，比如，配合[HSTS](https://developer.mozilla.org/en-US/docs/Glossary/HSTS)，它的"[Strict-Transport-Security](https://hstspreload.org/
https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)"可以防止man-in-the-middle攻击。关于更多CSP的介绍超出了本文范围，您可以查阅[这里](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).

#### X-XSS-Protection
该标头旨在防止XSS，但具有讽刺意味的是，它可以被用于XSS。 参见[Stackoverflow](https://stackoverflow.com/a/57802070/2559798)。 旧版浏览器的默认值为1，这会造成一个漏洞。 现代浏览器直接忽略了此标头。 如果要保护运行旧版浏览器的客户，则可能需要设置“ X-XSS-Protection:0”，并依赖于本文中介绍的其他机制来防止XSS。

## CSRF

浏览器具有同源策略（same-origin）以防止一个站点恶意操纵另一个站点。 在网站A上运行的脚本无法通过javascript将POST XMLHTTPRequest发送到网站B。但是same-origin policy存在某些特殊情况可以用来攻击：

### 问题1：4个特殊标签的GET请求允许cross-origin

与XSS不同，在CSRF中，黑客无法在受害者的浏览器中运行任意脚本，相反，黑客必须将受害者带到带有4个标签（img，link，script和iframe）的恶意网页，然后偷偷摸摸伪造用户请求发送到受害者的网站进行攻击。

假设受害者网站上的API如下所示： 

```
GET https://honest-website.com/pay?toAccount=<account>
```

像这样的恶意网站页面上有一个img标签
```
<img src="https://honest-website.com/pay?toAccount=hacker_account">
```
如果黑客可以让用户在被攻击网站上登录，然后再去恶意网站，则付款请求将从受害者的浏览器上，从恶意网站的页面发送到被攻击网站。 并且在大多数情况下，诚实的网站无法区别这是否是用户的意图，并将对其进行正常处理。 尽管同源策略可以保护用户免受javascript脚本发起的ajax请求的攻击，但是这4个tag发起的GET请求不受保护。

### 问题2：表单提交允许cross-origin

在web发展的早期，开发人员认为世界是美好的，没有黑客，因此cross-origin POST默认被允许，并且许多系统都基于此功能构建。 如果我们有机会回到过去并禁用cross-origin POST并定义CORS标准来指定允许的cross-origin请求，那么许多CSRF漏洞将神奇地消失。 不幸的是，我们不能更改它来打破兼容性，否则互联网上大量的web应用将无法正常工作。

因此，如果您在恶意网站中有Form，并将POST / PUT / DELETE请求发送到诚实网站，则允许这样做！ 并且可以默默地完成, 比如：
```
<form name="myform" action="https://fintech.com/pay" method="post">...</form>
<script>document.form.myform.submit();</script>
```

但是，这对于现代应用程序可能不是问题，因为它们不使用Form来提交请求。 现代的Web API接受“ application/json”，而不是“ application/x-www-form-urlencoded”。 如果您上传视频，则现代API会接受“application/octet-stream”或“video/mp4”，而不是“multipart/form-data”，因此，现代Web API根本不接受任何表单提交。 考虑到我们仍有一些遗留应用程序，在本文中我仍然会提出解决问题的建议，但是我要提醒您，当我谈论表单提交时，它仅适用于遗留应用程序。

请注意，如果您使用javascript从form创建REST payload，那么该浏览器将禁止该提交，因为它实际上是脚本发起，而不是简单用户发起。 例如，在jQuery中，这样的cross-origin操作还是会被禁止的：
```
  $.ajax({type: "POST", url: url,  data: form.serialize() });
```

### 防止CSRF攻击

#### 正确使用 HTTP Verb
在问题1的示例中，如果我们将HTTP动词更改为POST，则黑客将无法发起攻击，因为img标签无法发送POST请求。对于修改数据的接口使用POST/PUT/DELETE/PATCH而非GET 这是解决这个问题的最简单的办法。

但是对于不改变数据的API使用GET的时候，有时仍然很容易受到攻击。 例如，假设我们有一家金融科技公司为历史股价查询提供付费服务，而每个查询的费用为1美元。 黑客可能会创建一个包含数千个标签的页面，如下所示：
```
<img src="https://fintech.com/query-stock?ticker=SAP"/>
<img src="https://fintech.com/query-stock?ticker=GME"/>
...
```
一旦受害者打开此页面，这个金融科技公司的站点可能会向他收取数千美元。 换句话说，使用正确的HTTP动词只能保护进行数据更改的API，因为这些API不会使用GET请求，而对于没有数据更改的GET请求，某些场景下仍然有攻击价值，因此这个解决方案无效。 许多安全专家和论文建议对POST/PUT/PATCH/DELETE的保护请求，但这是不正确的。

无论如何，尽管此解决方案无法保护GET请求的API，并且该方法无法防止跨域表单提交（虽然很老的应用才会使用表单提交），但我仍强烈建议您这样做，因为遵循HTTP标准还有很多其他好处（比如客户端知道什么 API是幂等的，可以重试，但是这不在本文的讨论范围之内）。

### Referer Header 检查
您可以检查referer头是否完全来自您的网站，拒绝掉不信任的来源referer。这是因为“ 4个tag和form”只能发送带有标准header的请求，黑客无法操纵受害者的浏览器发送带有这4个tag或form的额外header。这很容易实现，但是有一些陷阱。如果本机iOS应用程序也使用该API，则可能没有referer头。并且某些网关或Load Balancer可能会在服务器接收该header之前将其删除。实际上，referer设计的很草率，它从未被设计为以非常准确的方式使用，更不用说连“referer”都拼错了(应该是referrer)。这个header还可能会随着不同的浏览器策略而发生变化，例如[chrome changes the default policy last year](https://developers.google.com/web/updates/2020/07/referrer-policy-new-chrome-default)，并且某些旧版浏览器不遵守该政策。我不建议将此header用于CSRF预防，因为它不准确。

#### SameSite Cookie 属性

SameSite是cookie属性的新标准（类似于HTTPOnly，Secure等）。它可以是三个值。

* Strict 防止在所有cross-site浏览上下文中浏览器将cookie发送到目标站点，即使点击一个普通链接也是如此。这有时会牺牲用户体验，但绝对是最安全的。
* Lax 表示仅当用户使用GET请求并导航到原始站点时（浏览器的URL要变化），才允许使用cookie。这包括点击链接或使用GET提交表单。新的浏览器默认为此。
* None：如果未指定，则某些旧的浏览器将默认使用它，并且cookie将在所有上下文中发送，包括图像，超链接和POST请求。显然这是非常不安全的。

>这里我们说的是cross-site，而不是cross-origin，因为基于历史原因cookie不是cross-origins的，cookie虽然可以指定domain，但是不能为一个域名下的每个端口设置cookie。有关更多详细信息，请阅读[跨域和跨站点](https://web.dev/same-site-same-origin/).

在大多数情况下，如果您的客户使用的是现代浏览器，而您的应用程序使用了正确的HTTP动词，那么Lax足以阻止CSRF。此解决方案的缺点是，某些旧的浏览器不支持此解决方案，你不能完全依赖于这个特性，所以请在使用此解决方案之前考虑[浏览器兼容性](https://caniuse.com/same-site-cookie-attribute)。

#### 自定义 Header
由于这些“ 4个tags和form”无法添加额外的header，因此您可以为所有请求添加一个特殊的header，并在服务器端验证该header。 例如，
```
"X-Requested-By: whatever value, supported by Jersey server"
```

如果您使用像axios这样的客户端，则可以将其设置为默认值，不用每次发请求都写代码。如下所示：
```
axios.defaults.headers.common['X-Requested-By'] = 'desktop-react-app';
```
这是非常简单有效的方案。

实际上，更进一步，您可以停止使用Cookie来保存会话ID，您可以仅将会话ID保留在浏览器本地存储中，然后通过XMLHTTPRequest使用特殊标头将其发送：
```
X-My-Auth: <your token here>
```
那么无论如何，您的会话ID将不受CSRF的影响，因为CSRF依赖于cookie。 如果您不使用Cookie，则无法利用CSRF相关的攻击向量。

#### CSRF Token
CSRF令牌可能是最广泛建议的解决方案。 CSRF令牌在服务器端生成，可以针对每个用户会话或针对每个请求生成一次。当然，每个请求一个令牌更安全，但出于多种原因，我强烈建议每个会话令牌或直接用HMAC令牌。

如果每个请求都生成一个令牌，存储和加载令牌的开销是巨大的浪费，对于大型Internet应用程序有时是不可能的。每个会话令牌好一些，因为它每个会话只生成一次，但是对于每个请求，它仍然需要一次读取令牌操作（例如通过Redis这样的存储）。为了减少I/O操作，我们可以使用HMAC令牌实现令牌，然后可以通过算法生成和验证令牌，而无需任何存储。您只需确保HMAC密钥已固定并每月轮换一次即可。

我不建议使用CSRF token，因为它最适合使用表单提交的旧应用程序。 <b>读到这篇文章，您一定会感到震惊，并认为我疯了</b>。一点也不！让我解释一下原因。当您使用某些古老的模板引擎（例如JSP或Thymeleaf）渲染页面时，可以在表单的隐藏字段中生成每个请求令牌，然后提交内容类型为“ application/x-www-form-urlencoded”的表单。但是，在现代应用程序中，我们不从服务器端的模板引擎渲染表单，我们必须<b>首先请求获得令牌</b>，然后将其置于React状态，而不是隐藏的表单字段，然后使用fetch或axios之类的客户端把form转为JSON再提交请求，令牌这时候是在header上的，而不是一个form field。这样每次调用后端都要先发个请求获取令牌，有时候您对后端的请求会加倍，这是丑陋的额外工作。对于没有表单提交的现代应用程序，您可以使用其他更简单的解决方案，不要再不明就里的用CSRF token了。

#### CORS 配置导致的漏洞
到目前为止，我们知道有两个问题需要您注意以防止CSRF，所有其他情况都受到浏览器同源策略的很好保护。 但是有时您还想要跨域访问，这就是为什么会有[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)。CORS的功能非常强大，可帮助您将Web应用程序集成在一起并相互通信，但是如果配置错误，它也会带来严重的CSRF问题。 例如， 以下政策允许任何网站将请求发送给您的网站并附带Cookie, 您的网站一定会悲剧的。
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
另一个问题是，CORS会导致XSS传递攻击。如果您信任并允许某个网站使用Cookie来访问您的网站（将Access-Control-Allow-Credentials设置为true），如果该网站具有XSS漏洞，<b>则黑客可能会从您信任的网站上先注入XSS，然后从您信任的网站再通过CSRF攻击您的网站</b>。

## 结论

### XSS 建议
使用现代Web框架，请避免使用诸如“ dangerouslySetInnerHTML”之类的特殊功能，不要转义用户输入，仅在呈现时转义它。

### CSRF 建议
<b>将正确的HTTP Verb和自定义Header与现代Web框架配合使用在大多数情况下就足够了</b>。 CSRF令牌是使用最广泛的解决方案，但我不建议您这样做，因为它比其他解决方案复杂得多，只有在您的遗留应用程序具有表单提交功能时才使用它。 启用CORS时，您可能会受到信任的网站的攻击。
