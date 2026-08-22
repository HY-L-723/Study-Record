## JWT

#### JWT 本身不是登录上下文

JWT 是客户端携带的字符串。Spring Security 不会因为请求头里出现三个点分段，就自动承认用户身份。应用必须验证签名和有效期，然后显式建立认证对象

#### JWT验证的两道门

- 签名门：用服务端密钥重新验证 HMAC。payload 任意一位发生改变，原签名都不匹配。
- 时间门：当前时间必须早于 exp。签名正确只说明内容没被改，不说明凭证仍有效。

 #### 过滤器链中的交接

 Bearer JWT → JWT 过滤器 → Authentication → SecurityContextHolder → 授权过滤器 → Controller

- 登录：你在前台出示身份证，前台核对成功后发给你一张门禁卡。对应提交用户名、密码并取得 JWT。
- 认证 Authentication：门禁系统刷卡后确认“这是 Alice 的有效卡”。它回答的是：你是谁，凭证是否有效。
- 授权 Authorization：系统已经知道你是 Alice，再判断 Alice 能不能进入机房。它回答的是：你能做什么。

#### 401 和 403
- 401 未认证：没带卡、卡是假的、卡已过期。系统没有一个当前有效身份。
- 403 无权限：卡有效，系统知道你是谁，但你的权限不允许进入目标区域。

#### JWT三段结构
- Header
说明怎么处理这个 token。

{"alg":"HS256","typ":"JWT"}
alg 是签名算法，typ 是类型。

- Payload
保存关于用户和 token 的声明（claims）。

{"sub":"alice","exp":...}
sub 常表示用户身份，exp 表示过期时间。

- Signature
服务端使用密钥对前两段计算出的签名。

HMAC(key, h64 + "." + p64)
用于发现 header 或 payload 是否被修改

#### 服务端的验证过程
- 客户端发送 header、payload、signature。
- 服务端用自己的密钥，根据收到的 header 和 payload 重新计算签名。
- 新计算的签名与 JWT 第三段比较。
- 不一致就说明内容或签名不可信，认证失败

#### 让 Spring Security 认识当前用户
JWT 验证成功，只表示应用现在拥有可信的用户信息。接下来还要把这些信息交给 Spring Security 的标准认证模型。

Authentication：身份档案
表示当前主体是谁、拥有哪些权限、是否已经认证。示例中可能包含用户名 alice 和权限 ROLE_USER。

SecurityContext：本次请求的安全档案盒
主要保存当前的 Authentication。后续授权组件从这里取得身份和权限。

SecurityContextHolder：访问档案盒的统一入口
过滤器通过它放入认证结果，Controller、Service 和授权过滤器也通过它读取当前认证上下文。

有效 JWT → 创建 Authentication → 放入 SecurityContext → 通过 SecurityContextHolder 保存/读取

#### 代码示例
- 提取凭证
```
String authorization = request.getHeader(HttpHeaders.AUTHORIZATION);

if (authorization != null
    && authorization.startsWith("Bearer ")
    && SecurityContextHolder.getContext().getAuthentication() == null) {
```
含义：请求必须携带 Bearer Token，并且当前上下文还没有认证身份，过滤器才尝试 JWT 认证。避免无 token 时解析，也避免覆盖已经存在的认证。

- 验证并读取可信信息
```
Claims claims = jwtService.verifyAndRead(authorization.substring(7));
```
substring(7) 去掉 Bearer 。

verifyAndRead 验证签名和过期时间；成功返回可信 claims，失败抛异常。

- 把 JWT 身份交给 Spring Security
```
var authentication = UsernamePasswordAuthenticationToken.authenticated(
    claims.getSubject(), null,
    List.of(new SimpleGrantedAuthority("ROLE_USER")));

SecurityContextHolder.getContext()
                     .setAuthentication(authentication);
```
subject 作为当前用户名，权限为 ROLE_USER。创建已认证对象并写入上下文后，后续 authenticated() 才能放行。
- 继续链路
```
chain.doFilter(request, response);
```
把请求交给后面的授权过滤器。JWT 过滤器不是最终业务处理者，也不能认证成功后直接跳过后续链。
- 失败路径
```
catch (JwtException | IllegalArgumentException ignored) {
    // 不写入 Authentication
}
```
受保护接口随后因上下文没有有效认证而统一返回 401；公开登录接口仍可通过。

#### 登录链

用户名密码 → 校验账号 → 创建 JWT → 使用密钥签名 → 返回 token
- 第一步：登录接口必须公开
```
.requestMatchers("/auth/login").permitAll()
.anyRequest().authenticated()
```
用户还没有 JWT 时也必须能够调用登录接口。其他接口默认要求已经认证。

- 第二步：先校验密码，再签发 token
```
if (用户名错误 || 密码不匹配) {
    return 401;
}
return jwtService.issue(username);
```
JWT 不能替代首次登录的密码校验。只有账号凭据验证成功后，服务端才应该为该用户签发 JWT。

- 第三步：构造三段 JWT
```
return Jwts.builder()
    .header().type("JWT").and()
    .subject(username)
    .issuedAt(Date.from(now))
    .expiration(Date.from(now.plus(ttl)))
    .signWith(key)
    .compact();
```
type("JWT")：header 中的 typ。

subject(username)：payload 中的 sub。

issuedAt、expiration：payload 中的 iat、exp。

signWith(key)：使用服务端密钥生成 signature。

compact()：输出 header.payload.signature 字符串。

签发与校验是一对
issue 使用密钥生成签名；后续 verifyAndRead 使用对应密钥验证签名。HMAC 场景中两端是同一个服务端秘密密钥，因此绝不能把密钥发给客户端或写入 payload

## SSE 流式传输与会话隔离
- 最小合法事件

Content-Type: text/event-stream

data: hello

空行触发事件派发；正文结束本身不替代空行。

- 常用字段

data:：事件数据

event:：自定义事件名

id:：断线重连使用的事件 ID

retry:：建议重连间隔（毫秒）

: 开头：注释，可作心跳

- 多行数据

data: first
data: second

产生一个事件，event.data 为 first\nsecond。

- FastAPI 心智模型

async generator
  
    → yield "data: ...\n\n"
  
    → StreamingResponse
  
    → HTTP response chunk
  
    → EventSource parser

yield 负责给出块；字符串内容仍须遵循 SSE 帧格式

#### 四个字段的职责

- generate_events()：决定业务数据按什么顺序产生。
- yield：把当前字符串交给响应层，然后保留生成器状态，之后从原位置继续。
- StreamingResponse：迭代生成器，把各块持续写入同一个 HTTP 响应。
- EventSource：按 SSE 规则解析文本，看到空行才派发事件

#### Redis key
sse:{user_id}:{session_id}

user_id 必须来自已认证身份，不能只信请求参数。每个 key 设置 TTL，并在读写前校验会话归属。

#### 症状 → 首查
- 迟迟没有事件：缺少 \n\n 或被代理缓冲
- 所有内容最后一起到：阻塞生成器或中间层缓冲
- 用户串话：key 缺用户/会话维度
- 缓存不断增长：未设置 TTL 或清理
- 重复内容：重连后未处理事件 ID/游标

# InnoDB 索引速查

## 叶子记录

| 索引                     | 叶子层核心内容      | 查整行                   |
| ------------------------ | ------------------- | ------------------------ |
| 聚簇索引（通常 PRIMARY） | 整行记录            | 直接获得                 |
| 二级索引                 | 二级索引列 + 主键列 | 通常再查聚簇索引，即回表 |

**覆盖索引**不是一种特殊索引类型，而是某个索引恰好包含该查询需要的全部列。二级索引中的主键列也可参与覆盖。

## 联合索引 `(a,b,c)`

| 条件                    | 快速判断                                                                       |
| ----------------------- | ------------------------------------------------------------------------------ |
| `a=?`                 | 使用左侧前缀 a                                                                 |
| `a=? AND b=?`         | 使用 a,b                                                                       |
| `a=? AND b=? AND c=?` | 使用 a,b,c                                                                     |
| `b=?` 或 `c=?`      | 不能依赖传统最左前缀做直接定位；实际计划由优化器决定                           |
| `a=? AND b>? AND c=?` | a,b 通常确定连续扫描区间；c 仍可能做索引条件过滤或覆盖，不能笼统说“完全失效” |

## 四步判断

1. 按联合索引顺序，从最左列开始。
2. 连续等值条件通常可继续向右。
3. 第一个范围条件通常终止继续构造单一连续查找区间。
4. 再单独检查：后续列能否过滤、能否覆盖、能否帮助排序。

## 验证

查看 `EXPLAIN` 的 `key`、`key_len`、`type`、`rows`、`Extra`；有环境时优先用 `EXPLAIN ANALYZE` 对照实际行数和耗时。

来源：MySQL 8.4 Reference Manual，Clustered and Secondary Indexes；How MySQL Uses Indexes；EXPLAIN Output Format。
