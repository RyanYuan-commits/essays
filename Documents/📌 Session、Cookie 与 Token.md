Session、Cookie 和 Token 都是为了解决在无状态的 HTTP 协议上，如何识别访问者身份的问题而提出的方案。核心是 **存储**。
# Cookie
## Cookie 简介
实现浏览器每次请求都自动携带参数的技术。
基本执行流程：
浏览器发送 HTTP 请求，服务器会进行 Cookie 配置，也就是 `Set-Cookie`，Cookie 有 Name 和 Value 两个重要属性。
浏览器接收到这个 Cookie 的时候，会将其持久化，之后每次发送请求的时候都会携带这个 Cookie。
在浏览器中可以很容易的看到明文存储的 Cookie，所以将密码存储在 Cookie 是很不安全的。
![[在浏览器中查看 Cookie.png]]
## Cookie 的各种属性
Name 和 Value：
	**Name** 是Cookie的名称，Cookie一旦创建，名称便不可更改，**一般名称不区分大小写**；
	**Value** 是该名称对应的Cookie的值，如果值为Unicode字符，需要为字符编码。如果值为二进制数据，则需要使用BASE64编码。
Domain:
	Domain 决定 Cookie 在哪个域是有效的，也就是决定在向该域发送请求时是否携带此 Cookie，Domain 的设置是对子域生效的，如Doamin设置为 .a.com,则b.a.com和c.a.com均可使用该Cookie，但如果设置为b.a.com,则c.a.com不可使用该Cookie。Domain 参数必须以点(“.”)开始，Domain 也不区分大小写。
Path:
	Path是Cookie的有效路径，和Domain类似，也对子路径生效，如Cookie1和Cookie2的Domain均为a.com，但Path不同，Cookie1的Path为 /b/,而Cookie的Path为 /b/c/,则在a.com/b页面时只可以访问Cookie1，在a.com/b/c页面时，可访问Cookie1和Cookie2。Path属性需要使用符号“/”结尾。
Expires/Max-age:
	Expires和Max-age均为Cookie的有效期，Expires是该Cookie被删除时的时间戳，格式为GMT,若设置为以前的时间，则该Cookie立刻被删除，并且该时间戳是服务器时间，不是本地时间！若不设置则默认页面关闭时删除该Cookie。
	Max-age也是Cookie的有效期，但它的单位为秒，即多少秒之后失效，若Max-age设置为0或-1，则立刻失效。Max-age默认为 -1。
	Max-age优先级高于Expires
Size
	**Szie**是此Cookie的大小。在所有浏览器中，任何cookie大小超过限制都被忽略，且永远不会被设置。
	各个浏览器对Cookie的最大值和最大数目有不同的限制，整理为下表(数据来源网络，未测试)：
# Session
将用户所有的登录信息存储在 Cookie 中显然是不太可行的，那将信息存储在服务器端呢？
这就是 Session 技术，用户的每个会话都会被一个 SessionID 所唯一标识。
基本执行流程：
浏览器发送 HTTP 请求，使用账户和密码进行登录。
服务器验证登录信息正确，在服务器持久化一个生成 Session，这个 Session 会有一个 SessionID 和一个 Max-age（过期时间）。
然后服务器将 SessionID、Max-age 以及一些其他信息，通过 Cookie 技术发送给浏览器。
之后浏览器的每次请求都会携带这个 SessionID，服务器就可以通过这个 SessionID 获取到服务器中存储的信息。
# Token
用于现在网站的用户量越来越大，如果将 Session 存储在服务器的内存中，服务器的存储压力会越来越大。
而如果有多台服务器，一个服务器存储的 Session 数据，其他服务器是无法获取的，此时又存在 Session 共享的问题（可以解决）。
但除了服务器之外，唯一能选择存储状态数据的位置就是浏览器了，而 Cookie 的明文存储性质又不太安全，所以将信息加密后存储在浏览器，这就是 Token 技术。
其中最著名的技术是 JWT 技术（Json Web Token）。
JWT 中存储的是加密过后的用户信息，服务器可以将 JWT 通过密钥加密后发送给浏览器，浏览器可以通过 LocalStorage 或 Cookie 存储在浏览器本地，之后发送请求的时候在 Header 中携带上这个 Token；
服务器收到 Token 后，利用密钥解密，就可以知道用户的身份了。