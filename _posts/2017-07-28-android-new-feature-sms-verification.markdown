---
layout: post
title:  "Android 8的安全新特性:下行短信验证"
date:   2017-07-28 17:33:00
published: true
---

# 前言

目前通过下行短信验证码来验证手机号码来完成注册是非常常用的手段，尽管他的安全性存在一定问题，但是下行短信验证在短期内仍然是在安全和体验方面比较比较平衡的选择。

(去年NIST已经不再推荐下行短信两阶段验证，今年运营商O2 Telefonica在德国已经出现了通过SS7漏洞对银行的二次短信验证攻击成功并转账的案例)

尽管下行短信已经很方便了，但是我们还是想让这个流程更加流畅，那就是避免用户手动输入或者复制短信中的验证代码。不过这样就得增加READ_SMS的权限然后读取用户的短信，这其实有很大的安全问题。首先，很多用户对于短信读取权限是很敏感的，另外，如果每个应用都有读取短信的权限，那么这些应用可以同时读取用户的短信，这就会造成短信验证码被恶意的应用盗取。

# 原理

为了解决这个体验和安全两方面的问题，这个流程在android 8中得到了很好的优化，用户整个过程不再需要手动输入或者复制短信验证码，也不需要应用增加READ_SMS权限。具体的做法如下：

<img src="/images/2017-07-28/sms_verification_seq.png" max-height="500px">

上面这个时序图我们可以看做四大步

第一步(消息1,2,3)，客户端先通过SmsManager的createAppSpecificSmsToken方法来创建一个一次性的token, 操作系统记录这个token是哪个app发起的，然后把这个token和手机号提交给服务器。

第二步(消息4,5,6)，服务器把生成有效期为几分钟的code，然后把手机号和token映射到这个code，然后向短信网关发短信，内容为"token + code"。

第三步(消息7,8)，客户端操作系统接收到这条短信后，识别出token，然后通过包含短信内容的Intent通知所对应的应用，这个过程其它应用不会收到通知并且这条短信也不会进入短信信箱。

第四步(消息9,10,11)，客户端收到Intent后向服务器提交code，完成验证。


需要注意的是，这个过程中用户全程没有文本操作，所以体验更加流畅。另外，这条短信不进入信箱，其他具有READ_SMS权限的应用无法读取到这个短信，而且操作系统只会把内容发给token对应的应用，所以会比READ_SMS的方式要安全很多。目前Android 8还是preview阶段，所以这个token是不会过期的，只是每次生成新token都会让老token失效，等Android 8正式发布的时候这个token的有效期可能会限定很短。

# 测试
由于目前还没有Android 8的设备，我们需要下载最新版本的Android O Preview的虚拟机，通过虚拟机来测试。

我们只需要在一个Activity中调用以下代码即可生成token。
{% highlight java linenos %}
PendingIntent intent = PendingIntent.getActivity(this, 0,
    new Intent(this, SmsTokenResultVerificationActivity.class), 0);
String appSmsToken = SmsManager.getDefault().
    createAppSpecificSmsToken(intent);
{% endhighlight %}

为了测试，我们跳过服务器部分，直接模拟服务器调用短信通道网关成功，所以你需要在模拟器中向虚拟机发送一条短信，内容为appSmsToken和假设是服务器生成的code，中间用空格隔开。(当然，你也可以用其它格式, 比如json)

这时候你会发现SmsTokenResultVerificationActivity这个Activity被唤起了，你可以在这个Activity的onCreate里加入日志, 或者显示到界面上来验证短信内容：

{% highlight java linenos %}
TextView view = findViewById(R.id.result);
for (SmsMessage pdu : Telephony.Sms.Intents.
    getMessagesFromIntent(getIntent())) {
    String message = pdu.getDisplayMessageBody();
    Log.i("SMS message", message);
    view.append(message);
}
{% endhighlight %}

你会看到app原封不动地收到了短信内容，之后应用就可以解析短信内容，向服务器提交code来验证了。

# 总结
由于市面上大量的设备短期内都不支持api level 26，安卓设备的碎片化又比较严重，我们短期内还是需要同时兼容老版本设备，但是对于新设备，这将是一个非常棒的安全和体验的改善。
