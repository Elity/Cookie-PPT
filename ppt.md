name:inverse
class: center, middle,inverse
layout: true

---

# Cookie 那些事儿

Fighting

???

旨在让所有人对 cookie 有个全面系统的了解，同时对 web 安全的认知上一个台阶。
很多人只知道 document.cookie，关于 cookie 的其它一无所知。

---

- 你所知道的 cookie 是什么
- http 协议是否有状态
- 你如何看待 cookie 的安全问题
- cookie 和 session 是否有关系
- cookie 与 webstorage 有什么异同

???

1.简单描述你所认知的 cookie

2.服务端是如何区分不同用户的

3.如果仅仅只是使用 document.cookie="a=1"，毫无安全可言，那为什么 webstorage 相对安全呢？ [csrf demo](./demo/csrf.html)

4.禁用浏览器的 cookie 功能，是否影响服务端 session 的工作

5.webstorage 的出现能否取代 cookie？

- webstorage(具体来说是 localstorage)有事件 storage event [storage event demo](./demo/storage-event.html)
- localstorage 遵循同源策略，而 cookie 限制会更多
- sessionstorage 只针对单个会话(标签)
- webstorage 有专门的增删改查 api
- webstorage 可用容积要大得多，cookie 只有 4k 左右，同时还有数量限制（现代浏览器为 50 左右）
- webstorage 无法被服务端控制，而 cookie 可以
- cookie 会自动附加在 http 请求中
- cookie 可以设置自动过期时间

---

## 读取 cookie

???

开胃小菜。

```javascript
console.log(document.cookie);
```

思考：

- 是不是所有的 cookie 都可以被上面这个代码读取？ (httpOnly 标识的不能读取)
- a.com 网站下能否读取 b.com 网站下的 cookie？ 为什么不能？ 同源策略限制么？(不能，非同源策略)
- 有没有办法知道某个 cookie 什么时候过期？ （很遗憾，浏览器暴露的 cookie 操作方式只有 document.cookie）

---

## 设置 cookie

???

这才是主菜，取 cookie 简单，设置 cookie 的幺蛾子很多

---

## 格式

---

```
<cookie-name>=<cookie-value>;
Max-Age=<non-zero-digit>;
SameSite=<Strict|Lax>;
Domain=<domain-value>;
Path=<path-value>;
Expires=<date>;
HttpOnly;
Secure;
```

???

HttpOnly 比较特殊，只能通过 HTTP 方式来设置

---

## cookie-name=cookie-value

???

从基本的开始，设置 kv

```javascript
document.cookie = 'a=1'; // 成功
```

```javascript
document.cookie = 'a=2;b=3'; // a更新成功，b添加失败
```

必须出现在一条设置 cookie 语句的最前面。一次只能设置一个。

[RFC6265](http://www.ietf.org/rfc/rfc6265.txt)对于 cookie-name 与 cookie-value 有一些规范上的约束，但是各家浏览器实现了更多的支持，但又不统一，但可以肯定的是`;`是不能用的。

为了防止出现诡异的错误，尽量设置符合标准的 cookie。

一些特殊的 cookie-name 稍后介绍

---

## Expires=date

???

用户设置 cookie 的过期时间，格式类似于：

```javascript
new Date().toUTCString();
```

不明白具体规则的情况下，不要尝试手写 date，在 [RFC6265](http://www.ietf.org/rfc/rfc6265.txt) 搜： 5.1.1.

不设置过期会怎样？

不设置过期时间的话，cookie 就是会话级别，浏览器关闭则删除

---

## Max-Age=non-zero-digit

???

比 Expires 更人性化的设置 cookie 过期时间的方式，但是不兼容 IE9 以下浏览器。
在支持 max-age 的浏览器上同时设置 max-age 与 Expires，max-age 的优先级更高

MDN 上写了值为 non-zero-digit，但是根据 rfc 中的说明，取值可以是任意整数。
而实际验证是类似于对给定的值做 parseFloat 运算，然后设置
也就是说 max-age=5a;也是可以的

---

## Path=path-value

???

指定一个 URL 路径，这个路径必须出现在要请求的资源的路径中才可以发送 Cookie 首部。字符 %x2F (“/“) 可以解释为文件目录分隔符，此目录的下级目录也满足匹配的条件（例如，如果 path=/docs，那么 “/docs”, “/docs/Web/“ 或者 “/docs/Web/HTTP” 都满足匹配的条件）。

另外，由于 domain 与 path 是分开解析的，所以 a=1;domain=.a.com;path=/x/y,也能在 sub.a.com/x/y 下被读取。

---

## Domain=domain-value

???

开始比较复杂的几个属性

1.如果 domain 忽略会怎样？ （被标记为 hostonly，domain 项不以"."开头，a.com 下设置的无法在 sub.a.com 下读取)

2.domain 能包含端口号么？ （不能，domain 符合 hostname 规则，注意区别 location.host 与 location.hostname，这也说明了 cookie 不遵循同源策略）

3.domain 设置有什么限制么？

演示[fighting.com](http://fighting.com)

- sub.a.com 可以设置 .a.com (是否说明二级域名一定可以设置一级域名？)
- a.com 不能设置 sub.a.com
- sina.com.cn 不能设置 .com.cn
- w3c.github.io 不能设置 .github.io

Mozilla 很早之前就为此建立了一个列表[Public Suffix List](https://publicsuffix.org/list/public_suffix_list.dat)，
主流浏览器都在使用这个列表 [官网介绍](https://publicsuffix.org/learn/)

如果希望自己的网站添加进该列表，可以来[这里](https://github.com/publicsuffix/list)提 PR（被通过应该需要很多手续）。

---

## SameSite=Strict|Lax|Unset(默认)

???

先看个[示例](./demo/samesite.html)

目前[兼容性](https://caniuse.com/#search=samesite)不太理想的一个属性

可以有效防御跨站请求伪造攻击(CSRF)。

设定 cookie 在跨域的情况下是否需要发送到服务器(禁读不禁写)

- Strict 任何时候都不跨域发送 cookie，所以当你发现从 a 网站进入 b 网站，b 网站的登陆状态总是失效的，然后刷新一下又正常了，不要悲伤、不要诧异，检查下你用于标记登陆状态的 cookie 是不是设置了 samesite 为 strict
- Lax 点击 a 标签、form 的 get 请求、通过 location 改变、通过 window.open 方式打开时会携带 cookie，而 ajax、script、link、img、iframe、form 的 post 发起的跨域请求则不会携带 cookie
- Unset 默认值，任何时候都会跨域携带 cookie

---

## Secure

???

这个属性有什么用？防范什么安全问题？

一个带有安全属性的 cookie 只有在请求使用 SSL 和 HTTPS 协议的时候才会被发送到服务器，同时无法在非 https 的页面通过 document.cookie 读取。这可以有效防范[SSl strip](https://github.com/Elity/sslstrip-nodejs)后 cookie 失窃。然而，保密或敏感信息永远不要在 HTTP cookie 中存储或传输，因为整个机制从本质上来说都是不安全的，比如前述协议并不意味着所有的信息都是经过加密的。
新版 Chrome 与 Firefox 已经`不支持`在非 https 的链接中设置 Secure 了。

演示通过 SSLstrip 窃取 baidu 的 cookie:
开启代理模拟被劫持，访问 evil.baidu.com

---

## HttpOnly

???

带此标识的 cookie 只能以 http 的方式设置于读取，也就是说，通过 document.cookie 的方式既不能写入也不能读取带 HttpOnly 标识的 cookie，可有效防止 xss 的方式窃取 cookie。事实上，一般后端 web 框架的 sessionid 一般都是设置了 HttpOnly 的。

---

## 第三方 cookie

???

访问`a.com`的时候，由于引入`b.com`的资源（img、script、iframe、link），该资源通过 http 的`set-cookie`头向浏览器写入 cookie，这个 cookie 虽然写在`b.com`下，但对于用户来说是其在访问`a.com`时写入的，所以把这个 cookie 称为第三方 cookie。

判断策略不是同源，而是[Public Suffix List]

有什么用？

- 单点登录，淘宝与天猫
- 广告精准投放

当然，默认情况下，浏览器是不允许 ajax 请求写入第三方 cookie 的，同时也不让请求携带第三方 cookie。

如何解决：

- [xhr](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/withCredentials)

- [fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch#%E5%8F%91%E9%80%81%E5%B8%A6%E5%87%AD%E6%8D%AE%E7%9A%84%E8%AF%B7%E6%B1%82)

- [CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials)

浏览器启用：阻止第三方 cookie 后，`sub1.a.com`可以向`sub2.a.com`写 cookie，但无法向`b.com`写。（此处的写包括了 http 方式与 js 方式）。所以 `taobao.com` 登陆后，`tmall.com` 自动登陆的功能将会失效。

[3rd-party-cookies](https://blog.zok.pw/web/2015/10/21/3rd-party-cookies-in-practice/)

[p3p](https://www.w3.org/P3P/)

---

```
    console.log("Thanks!");
```
