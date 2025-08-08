### 1 什么是 JWT？
JWT（JSON Web Token）,是一种开放的标准，它定义了一种紧凑且独立的方式，用于在各方之间以 JSON 对象的形式安全的传递信息，是一种 **轻量级的**、**自包含的**、**可验证的** 令牌，通常用于 **身份验证** 和 **授权机制。**
#### 1.1 JWT 的特点
- **轻量级**：JWT 采用紧凑的 JSON 格式进行数据表示，具有简洁，数据量小的特性；但随着携带数据量的增大，JWT 的大小仍然会不可避免地增长，会占用一定带宽。
- **自包含**：JWT 将用户相关的信息，如用户身份、权限等，包含在一个独立的令牌中，这些信息被编码在 JWT 的有效载荷（Payload）部分，使得 JWT 在传递过程中不需要依赖其他外部资源或服务来获取额外的用户信息。
- **可验证**：JWT 使用数字签名来保证数据不会被篡改，且包含过期时间，可以对其有效期进行验证。
#### 1.2 JWT 使用场景
- **身份验证场景**：JWT 可以用作认证机制，允许用户在登录后获取系统生成的令牌，然后后面继续请求服务的时候将该令牌放在请求头中用于后续的请求，服务器可以验证 JWT 的签名，从而确认请求的发起者是经过身份验证的用户，比如可以通过解密来拿取我们放入载荷中的用户信息。这避免了在每个请求中都需要重新验证用户的凭据。
- **授权机制**：JWT可以包含用户的权限信息或角色信息，允许服务器在每个请求中使用这些信息来控制对资源的访问。因为JWT的信息可以被解码，服务端可以轻松获取其中的授权信息。
### 2 JWT 的结构
这里展示的是一个 JWT，由三部分组成：头部 Header、载荷 Payload、签名 Signature，中间使用 `.` 隔开：
![[JWT 密钥案例.png|center|700]]
#### 2.1 头部 Header
头部通常由两部分组成，包含 **类型** 和使用的 **签名算法** 信息，通常，它的结构是一个 JSON 对象，例如：
```json
{
  "alg": "HS256",   // 签名算法，例如 HMAC SHA-256
  "typ": "JWT"      // 令牌类型
}
```
#### 2.2 载荷 PayLoad
载荷包含有关声明 Claims 的信息，声明是载荷中包含的一系列键值对，也就是 JWT 携带的信息；
官方定义了三种类型的声明，分别是：注册声明、公共声明和私有声明：
- 注册声明：是一组预定义的声明，具有特定的名称和含义，比如 `iss` 表示令牌发行者、`exp` 令牌过期时间等；
- 公共声明：由使用者自定义的，用于在不同的应用程序或系统之间共享特定的信息，我们使用到的就是这一部分；
- 私有声明：为特定的应用程序或组织内部使用而定义的声明。这些声明只在特定的环境中具有意义，不打算被其他外部系统理解或使用。
#### 2.4 签名 Signature
验证消息完整性的数字签名部分，签名的算法在头部中被指定，例如，服务端使用 `HMAC SHA-256` 算法给 JWT 签名：
```java
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```
后续再用密钥和接收到的数据重新计算 `HMAC`，就可以验证数据有没有被篡改。
### 3 JWT 使用案例
引入 Maven 坐标
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

生成 JWT
```java
void jwtTest() {
	// 设置位于头部的签名算法
	SignatureAlgorithm signatureAlgorithm =SignatureAlgorithm.HS256;
	String secretKey = "jwtTest";
	// 利用 claim 来设定私有声明
	Long id = 1L;
	Map<String, Object> claim = new HashMap<>();
	claim.put("userId", id);
	JwtBuilder builder = Jwts.builder()
	.setClaims(claim)
	.signWith(signatureAlgorithm, secretKey.getBytes(StandardCharsets.UTF_8));
	String compact = builder.compact();
	System.out.println(compact);
}
```
输出结果为：
```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.91B00ttDwbLvDkm_bB5TKD8ZowWDM48xGAx468yqYQA
```

解密密钥
```java
Claims body = Jwts.parser()
	.setSigningKey(secretKey.getBytes(StandardCharsets.UTF_8))
	.parseClaimsJws(compact)
	.getBody();
System.out.println(body.get("userId"));
```