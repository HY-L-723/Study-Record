## 收尾
> MyBatis 的 update / delete 返回 int，通常不是返回“用户 id”，而是返回“影响行数”。
例如返回 1 表示修改或删除了 1 行，返回 0 表示没有任何记录被修改或删除。

### Service 层检查数据库中的unique约束业务逻辑目的
>为了提前给用户友好的业务错误。
数据库 UNIQUE 约束是最终兜底，尤其能防止并发请求同时通过 Service 检查后插入重复 username。

### Controller 层不推荐直接调用Mapper
>Controller 不推荐直接调用 Mapper，不是因为 Controller 一定会出错，而是因为职责混乱。
Controller 负责 HTTP 入参和响应，Service 负责业务规则和异常转换，Mapper 负责数据库访问。
如果 Controller 直接调 Mapper，业务校验、事务控制、异常处理会散落在接口层，后续难维护、难测试、难复用。

### @Options(useGeneratedKeys = true, keyProperty = "id")
>该字段可以回填自增id，可以将数据库自动生成的id回填到传来的User对象中，如果需要返回完整的字段信息，根据回填的id，可以在数据库中重新查询一下用户信息

### 参数校验-BeanValidation

#### DTO 软件包中主要是对前端的接收信息进行参数校验，可以减少Service层中的部分代码

相关注解@NotNull @NotBlank @Min等，需要引入validation相关依赖
- @NotNull可以判断传入参数非空
- @NotBlank可以判断传入的内容非全空格字段，比@NotNull多了一个“  ”这种类型的判断
- @Min 可以设置字段最小值信息
- 上述字段类型均可以设置message信息给用户友好提示

当设置这些字段时，Controller层需要将传入参数设置@Valid注解表示自动校验触发器，如果没有就可能不会进行相关的校验

可以将@RequestBody中转换成的Java对象检查是否符合注解规则

## MyBatis 分页查询

#### 一般参数：
- page:查询第几页
- size:每页多少条
- limit:本次最多取多少条
- offset:跳过前面多少条后开始取，公式为offset=（page-1）*size
- total:符合条件的总条数
- records:当前页的数据列表
例子
```
page=1,size=10 -> offset=0, limit=10
page=2,size=10 -> offset=10, limit=10
page=3,size=10 -> offset=20, limit=10
```
在设置分页查询的标准返回类时可以这样设置：
```java
package agent.new_ai.ai.chat.usercurddemo.common;

import java.util.List;

public class PageResult<T> {

    private Integer total;
    private Integer page;
    private Integer size;
    private List<T> records;

    public PageResult(Integer total, Integer page, Integer size, List<T> records) {
        this.total = total;
        this.page = page;
        this.size = size;
        this.records = records;
    }

    public Integer getTotal() {
        return total;
    }

    public Integer getPage() {
        return page;
    }

    public Integer getSize() {
        return size;
    }

    public List<T> getRecords() {
        return records;
    }
}
```
其中`PageResult<T>`中的T表示泛型，表示可能返回数据的类型不确定
### 可能存在keyword关键字的查询最好使用动态查询

可以创建xml文件内部写入相关的查询语句，一般步骤
- 在本地配置文件中写入mapper文件夹的路径
  类似信息`mapper-locations: classpath:mapper/*.xml`
- 在资源配置软件包中创建的xml文件下写入相关代码

```
  <?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="Mapper接口地址">

    <select id="方法名" resultType="返回类型，如果是自己创建的类，需要标注想要类的地址">
        select *
        from users
        <where>有可能的where查询条件
            <if test="keyword != null and keyword !=''">
                username like concat('%',#{keyword},'%')
                or nickname like concat('%',#{keyword},'%')
            </if>
        </where>
        limit #{size} offset #{offset}
    </select>

    <select id="countPage" resultType="int">
        select count(*)
        from users
        <where>
            <if test="keyword != null and keyword != ''">
                username like concat('%', #{keyword}, '%')
                or nickname like concat('%', #{keyword}, '%')
            </if>
        </where>
    </select>

</mapper>
  ```
### Mapper接口内部注解版代码
```
@Select("""
        select *
        from users
        where (#{keyword} is null or #{keyword} = ''
               or username like concat('%', #{keyword}, '%')
               or nickname like concat('%', #{keyword}, '%'))
        limit #{size} offset #{offset}
        """)
List<User> findPage(@Param("size") Integer size,
                    @Param("offset") Integer offset,
                    @Param("keyword") String keyword);

@Select("""
        select count(*)
        from users
        where (#{keyword} is null or #{keyword} = ''
               or username like concat('%', #{keyword}, '%')
               or nickname like concat('%', #{keyword}, '%'))
        """)
Integer countPage(@Param("keyword") String keyword);
```
两者功能一致，都是按照keyword传入值来进行分页查询，未传入就执行对users表中的所有用户进行分页查询


## JWT基础
#### 登录认证相关概念
- 登录：用户提交 username/password，系统验证身份，并在成功后返回登录凭证。
- 认证：后端判断当前请求是谁发来的，是否能识别出用户身份。
登录是拿 token 的过程，认证是后续用 token 识别身份的过程
#### Token的作用
Token 是登录成功后后端签发给客户端的身份凭证。
客户端后续请求携带 token，后端通过校验 token 判断用户是谁，以及 token 是否有效。
token中通常会储存能识别用户的信息，比如userid、过期时间等，并可以通过签名防止被篡改
#### Authorization
Authorization 请求头专门用于携带认证凭证。
后端可以在拦截器里统一读取 Authorization，校验 token，不需要每个接口重复写认证逻辑。

#### 401和403状态
- 401：未认证。后端不知道你是谁，通常是没登录、没带 token、token 无效或过期。
- 403：已认证，但无权限。后端知道你是谁，但你不能访问这个资源。


#### JWT功能的简易实现步骤

- 引入依赖
- 创建JwtUtil
- 先创建拦截器类
- 注册拦截器类

##### 相关代码

拦截器类
```
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler) throws Exception {

    String authorization = request.getHeader("Authorization");

    if (authorization == null || authorization.isBlank()) {
        response.setStatus(401);
        return false;
    }

    if (!authorization.startsWith("Bearer ")) {
        response.setStatus(401);
        return false;
    }

    String token = authorization.substring(7);

    try {
        JwtUtil.parseUserId(token);
    } catch (Exception e) {
        response.setStatus(401);
        return false;
    }
    
    return true;
    }
}
```
注册拦截器
```
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final LoginInterceptor loginInterceptor;

    public WebConfig(LoginInterceptor loginInterceptor) {
        this.loginInterceptor = loginInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/me")
                .excludePathPatterns("/login");
    }
}
```

JwtUtil
```
package agent.new_ai.ai.chat.usercurddemo.util;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;

import javax.crypto.SecretKey;
import java.util.Date;

public class JwtUtil {

    private static final SecretKey KEY = Jwts.SIG.HS256.key().build();

    private static final long EXPIRE_TIME = 1000 * 60 * 60;

    public static String createToken(Integer userId) {
        return Jwts.builder()
                .subject(String.valueOf(userId))
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + EXPIRE_TIME))
                .signWith(KEY)
                .compact();
    }

    public static Integer parseUserId(String token) {
        Claims claims = Jwts.parser()
                .verifyWith(KEY)
                .build()
                .parseSignedClaims(token)
                .getPayload();

        return Integer.valueOf(claims.getSubject());
    }
}
```
其中KEY应该写入配置文件因为改代码每一次重启后端都会重新进行创建，这样即使token没过期也会验证失败
jwt内容
```
header：说明 token 类型和签名算法
payload：存用户信息，比如 userId、过期时间
signature：签名，防止 payload 被篡改
```
