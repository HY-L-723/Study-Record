## Redis 基础

```
对于读多写少的数据，同一个数据可能被接口频繁查询。
把它放进 Redis 后，重复查询可以直接从缓存读取，减少 MySQL 压力，提高响应速度。
```

#### Redis 和 MySQL 区别
- MySQL是关系型数据库，Redis是非关系型数据库
- MySQL 主要把数据持久化存到磁盘，适合保存核心业务数据
- Redis 主要把数据放在内存中，适合做高速缓存、计数器、临时状态。

#### 缓存命中/未命中
- 缓存命中：要查的数据在 Redis 中存在，直接返回，不查 MySQL。
- 缓存未命中：Redis 中没有数据，需要回查 MySQL；查到后通常再写入 Redis。

#### TTL
`TTL`是Time To Live，表示缓存还能存活多久，就是过期时间
基本不设置用不过期的原因：
1. 数据可能长期不一致。
2. Redis 内存会被越来越多的缓存占用。
3. 很多冷数据长期不被访问，却一直占内存。

#### 相关问题
1. Redis 查询快，是不是可以完全替代 MySQL？为什么？

>Redis 不能完全替代 MySQL。MySQL 是核心持久化数据源，负责保存真实业务数据；
Redis 更适合作为缓存层，用来提升热点数据读取性能。对于用户详情这类读多写少的数据，可以采用 cache aside 模式：先查 Redis，未命中再查 MySQL，查到后写入 Redis 并设置 TTL。
更新或删除用户时，通常先更新数据库，再删除对应缓存，避免接口继续返回旧数据。Redis 不能完全替代 MySQL。
2. 为什么读缓存时通常是“先查 Redis，未命中再查 MySQL”？
>因为我Redis中的数据都存储在缓存中，查询速度更快响应更快
3. 修改用户信息后，你觉得应该“更新缓存”还是“删除缓存”？为什么？
> - 先更新 MySQL
> - 更新成功后删除 Redis 里的 user:id 缓存
> - 下次查询时 Redis 未命中
> - 再查 MySQL 拿到最新数据
> - 把最新数据重新写入 Redis
这种缓存模式是`Cache Aside Pattern`，旁路缓存模式。
4. 如果 Redis 挂了，GET /users/{id} 接口应该直接失败，还是降级查 MySQL？为什么？

>降级查询MySQL，因为Redis只是查询优化策略让响应变快，即使Redis挂掉，MySQL中依然可以进行相应的查询操作

### 项目中配置Redis
1.在`pom.xml`文件中引入相关依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
没规定版本号的情况下可以这么写，会自动下载SpringBoot版本适应的相关版本

2.在配置文件`application.yml`中添加相关redis配置
```
spring:
  data:
    redis:
      host: 连接的数据库地址
      port: 相关端口
```
确认相关redis已启动

相关代码
```
private final StringRedisTemplate stringRedisTemplate;
private final ObjectMapper objectMapper;
```
相应的缓存逻辑该类代码需要处理JSON异常，应该写入try中
```
String key = "user:" + id;

// 1. 先查 Redis
String userJson = stringRedisTemplate.opsForValue().get(key);//按key查出来的Json序列
if (userJson != null) {
    return objectMapper.readValue(userJson, User.class);//将Json转换成User类
}

// 2. Redis 没有，再查 MySQL
User user = userMapper.findById(id);
if (user == null) {
    throw new BusinessException("用户不存在");
}

// 3. 查到后写入 Redis，设置 TTL
stringRedisTemplate.opsForValue().set(
        key,
        objectMapper.writeValueAsString(user),
        10,
        TimeUnit.MINUTES
);

return user;
```

#### 缓存穿透、雪崩、击穿 
- 缓存穿透：查询 Redis 没有、数据库也没有的数据，导致请求持续打到数据库。
- 缓存击穿：热点 key 过期后，大量请求同时访问这个热点数据，压力瞬间打到数据库。
- 缓存雪崩：大量 key 在同一时间过期，导致大量请求同时绕过缓存打到数据库。
- 缓存穿透可以用空值缓存、布隆过滤器、限流等方式缓解。
- 缓存击穿可以考虑热点 key 永不过期、互斥锁、提前刷新等方案。
- 缓存雪崩可以通过随机 TTL 避免大量 key 同时过期。

#### Redis INCR 限流的局限
1. 固定窗口边界问题：
   如果限制每分钟 10 次，用户可能在 20:45:59 请求 10 次，又在 20:46:00 请求 10 次，短时间内实际打出 20 次。

2. 不是严格平滑限流：
   INCR + TTL 更像固定时间窗口计数，不能像令牌桶、漏桶那样平滑控制请求速率。

3. 高并发下要注意原子性组合：
   INCR 本身是原子的，但 INCR 后再设置 EXPIRE 是两个动作。入门阶段可以接受，真实项目可用 Lua 脚本保证组合操作原子。

4. 单维度限流不够：
   只按 userId 限流可能挡不住换账号；只按 IP 限流可能误伤同一公司或宿舍网络。真实项目常组合 userId、IP、接口、租户等维度。

5. 分布式部署下要统一走同一个 Redis：
   如果每台应用机器自己在内存里计数，就无法全局限流。使用 Redis 是为了让多台服务共享同一份计数。

相关代码
```
//固定id，测试一分钟最多访问10次
public Long testLimit() {
    Long userId = 1L;
    String minute = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMddHHmm"));
    String key = "rate:ai-chat:" + userId + ":" + minute;

    Long count = stringRedisTemplate.opsForValue().increment(key);

    if (count != null && count == 1) {
        stringRedisTemplate.expire(key, 70, TimeUnit.SECONDS);
    }

    if (count != null && count > 10) {
        throw new BusinessException("请求过于频繁，请稍后再试");
    }

    return count;
}
```

## MQ入门

#### 同步调用 / 异步调用
- 同步调用：调用方发起请求后，一直等被调用方处理完，拿到最终结果才返回。
- 异步调用：调用方发起请求后，不等待最终处理完成，先拿到一个确认结果，后续任务由后台继续处理。

例子：

同步：
用户点“生成报告” -> 接口等待 30 秒 -> 返回完整报告

异步：
用户点“生成报告” -> 接口立刻返回 taskId -> 后台慢慢生成报告 -> 用户之后查 taskId 获取结果

#### 生产者 / 队列 / 消费者
- 生产者 Producer：把任务发送到队列的一方。
- 队列 Queue：暂存任务消息的地方。
- 消费者 Consumer：从队列中取出任务并真正处理的一方。

真实项目场景中一般的逻辑：

- Controller / Service 接收用户请求，创建报告任务，并投递消息 -> 生产者
- 内存队列 / RabbitMQ / RocketMQ 保存 taskId 消息 -> 队列
- 后台线程 / 消费者程序拿到 taskId，生成报告，更新状态 -> 消费者

#### 削峰填谷
削峰填谷：请求高峰时，先把大量任务放进队列，不让它们同时压到后端处理逻辑；后台消费者按照自己的处理能力慢慢消费。

相关例子：

一瞬间来了 1000 个生成报告请求。

如果同步处理，1000 个请求可能同时打爆模型服务。

如果用 MQ，接口先把 1000 个任务放入队列，消费者每秒处理 10 个，系统压力就变平稳。

#### 解耦
解耦：调用方不需要直接依赖处理方的具体执行过程。

例子解释：

提交接口只负责创建任务、发送消息。

消费者负责生成报告。

两边通过“消息”协作，不需要强绑定在一次接口调用里。

#### poll和take
- poll()
1.队列有数据：取出并返回

2.队列没数据：立刻返回 null
- take()
1.队列有数据：取出并返回

2.队列没数据：一直等待，直到有新任务进来

#### 自动消费者启动方法

```
public InterviewTask getTask(Long id){
    InterviewTask task = taskMap.get(id);
    if (task == null) {
        throw new BusinessException("任务不存在");
    }
    return task;
}

public void processTask(Long taskId){
    if (taskId == null) {
        return ;
    }
    InterviewTask interviewTask = taskMap.get(taskId);
    if (interviewTask == null) {
        return ;
    }
    interviewTask.setStatus("PROCESSING");
    try {
        Thread.*sleep*(3000);
    } catch (InterruptedException e) {
        Thread.*currentThread*().interrupt();
        interviewTask.setStatus("FAILED");
        return ;
    }
    String result = interviewTask.getJobName()+interviewTask.getQuestion()+"模拟生成报告";
    interviewTask.setResult(result);
    interviewTask.setStatus("DONE");
}

*//消费者启动方法*
@PostConstruct
public void startConsumer() {
    Thread thread = new Thread(() -> {
        while (true) {
            try {
                Long taskId = taskQueue.take();
                processTask(taskId);
            } catch (InterruptedException e) {
                Thread.*currentThread*().interrupt();
                break;
            }
        }
    });

    thread.setDaemon(true);
    thread.start();
}
```

## Docker入门
Docker 解决“环境不一致”和“部署复杂”的问题。

#### 镜像 Image 和容器 Container
- 镜像 Image：模板 / 安装包 / 菜谱
- 容器 Container：镜像真正运行起来后的进程 / 实例 / 做出来的菜
- 镜像本身不等于正在运行的服务。容器才是运行中的服务。

#### 一般操作步骤
- 打包 jar

  使用mvn命令或者在idea侧边栏clean，package打包
- 写 Dockerfile
```
  FROM eclipse-temurin:23-jre//本地项目构建的java版本

WORKDIR /app//压缩到app目录下

COPY target/user-crud-demo-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080//映射到8080端口

ENTRYPOINT ["java", "-jar", "app.jar"]
```
- docker build 构建镜像

  docker build -t user-crud-demo:1.0 .
- docker run 启动容器

  docker run --name user-crud-demo-app -p 8080:8000 user-crud-demo:1.0
- 用接口验证容器运行成功


#### docker容器连接本地数据库
docker run --name user-crud-demo-app -p 8080:8080 `
-e SPRING_DATASOURCE_URL="jdbc:mysql://host.docker.internal:端口号/数据库名?useSSL=false&serverTimezone=Asia/Shanghai" `
-e SPRING_DATASOURCE_USERNAME="名称" `
-e SPRING_DATASOURCE_PASSWORD="密码" `
user-crud-demo:1.0

## Linux相关命令
- pwd / ls / cd
- mkdir / cp / mv / rm
- cat / tail / grep
  
  tail -n 100 app.log：查看最后 100 行日志
  
  tail -f app.log：实时跟踪日志，服务运行时最常用

  grep "ERROR" app.log：搜索日志里的 ERROR
- ps -ef | grep java

  ps -ef：查看系统里正在运行的进程
  
  grep java：只筛选包含 java 的进程
- nohup java -jar app.jar > app.log 2>&1 &
  ```
  nohup：终端关闭后，程序继续运行
  java -jar user-crud-demo.jar：启动 Spring Boot jar
   > app.log：把正常输出写到 app.log
   2>&1：把错误输出也写到 app.log
   &：放到后台运行
  ```

lsof -i:8000：查看 8000 端口被哪个进程占用

netstat -tunlp | grep 8000：从网络监听端口里搜索 8000

## Nginx 入门
Nginx处理逻辑
```text
浏览器访问：http://example.com/api/users
Nginx 接到请求
Nginx 判断 /api/ 开头
Nginx 转发给：http://127.0.0.1:8000/users 或 http://127.0.0.1:8000/api/users
Spring Boot 执行 Controller 和 Service
```
#### 正向代理 vs 反向代理
- 正向代理：

  你访问外部网站，但中间先经过一个代理服务器。

  服务端看到的是代理，不一定知道真实用户是谁。

- 反向代理：

  用户访问 Nginx，Nginx 再转发给后面的 Spring Boot。

  用户不知道后端真实端口和真实服务地址。

#### 404、500、502
404：Nginx 或 Spring Boot 没找到对应路径。

500：Spring Boot 找到接口了，但 Controller / Service / Mapper 执行时报错。

502：Nginx 要转发给 Spring Boot，但 Spring Boot 没启动、端口错了、地址错了，导致连不上。


相关配置文件书写
```
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
    }
}
```
```
root：找本地静态文件。
proxy_pass：转发给后端服务。
proxy_pass 末尾不带 /：查询时保留 /api 前缀。
proxy_pass 末尾带 /：去掉 /api 前缀。
```
