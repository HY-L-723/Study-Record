## synchronized 与 AtomicInteger

对于没有进行synchronized等操作的count变量的共享元素进行修改操作的时候不具有原子性，多个线程进行读取操作时可能都会读到同一个旧值，所以最后只写回一次的有效新增，这样就会出现错误
### synchronized
>引入synchronized保证同步/互斥执行，它主要保护的一段代码，而不是变量本身
>所以说synchronized维护的方法不止只进行保护共享变量的修改
>任何有在多线程操作下有可能出现错误的情况都可以使用synchronized维护

### AtomicInteger

>AtomicInteger是支持原子操作的整数类，它内部的incrementAndGet方法支持原子性地+1，并返回加完后的值

### 两者的适用场景

- AtomicInteger适合：点赞数+1，访问次数+1，提问次数+1
- synchronized更适合：保护一段包含多个步骤的业务代码，库存扣减通常还要结合数据库事务、行锁、redis锁等

对于之前在main函数中使用final关键字维护count数组使其可以被main函数中写的Thread类调用这种情况可以将count改为外面的static int类型使用场景如下：
```
private static int count = 0;

    static class Thread1 extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                count++;
            }
        }
    }

    static class Thread2 extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                count++;
            }
        }
  }
```
这种方式就可以直接调用静态变量了，然后在后面实现main方法即可
使用synchronized的格式：
```
public class CounterDemo{
    private static int count = 0;

    private static synchronized void increment() {
        count++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                increment();
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                increment();
            }
        });

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();

        System.out.println(count);
    }
}
```
使用AtomicInteger版本：
```
import java.util.concurrent.atomic.AtomicInteger;

public class CounterDemo {
    private static AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                count.incrementAndGet();
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                count.incrementAndGet();
            }
        });

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();

        System.out.println(count.get());
    }
}
```

## JVM、类加载、OOM

### JVM
JVM负责类加载和运行.class字节码文件，其中.class字节码文件时javac编译器编译.java文件得到

### 类加载
类加载时JVM层面，JVM加载.class类信息，Spring会在这个基础上创建和管理Bean，而且在类初始化阶段，static代码块会被执行，一般只执行一次

### OOM
OOM全称`OutOfMemoryError`，主要原因是堆内存被占的无法放下新对象就会出现该报错，出现该错误的情况有可能是List中存储User对象，只要List还存在，不管User会不会被调用，List内部的User都不会被GC回收，所以堆内存占用越来越多最后爆满报错

### StackOverflowError
`StackOverflowError`主要是因为线程栈空间不够，典型原因是因为递归太深，方法调用链太深导致
例子：
```
public class StackOverflowDemo {
    public static void main(String[] args) {
        test();
    }

    public static void test() {
        test();
    }
}
```

### static 代码块和构造方法的区别
static代码块在类初始化时就会执行，通常只执行一次，构造方法是创建一个该类对象就会执行一次

## DTO
DTO = Data Transfer Object 是数据传输对象，用来接收请求或返回响应
在项目中实现例子：
1. 新建 CreateUserRequest
2. 字段：username、nickname、age
3. POST /users 改成 @RequestBody CreateUserRequest
4. Service 内把 CreateUserRequest 转成 User
5. username 为空时抛 BusinessException("用户名不能为空")
6. 成功时继续返回 Result.success
其中CreateUserRequest代码写入DTO中，用来接收请求内容
其中函数的代码块实现：

`AddUser`代码块：
```
其中nextId是静态变量，连接数据库后ID应该由数据库自增创建
public User addUser(CreateUserRequest createUserRequest){
        if(createUserRequest == null){
            throw new BusinessException("用户不能为空");
        }
        if(createUserRequest.getUsername() == null || createUserRequest.getUsername().trim().isEmpty()){
            throw new BusinessException("用户名不能为空");
        }
        if(createUserRequest.getAge() < 0 ){
            throw new BusinessException("年龄不能小于0");
        }
        User user = new User(nextId++,createUserRequest.getUsername(), createUserRequest.getAge(),createUserRequest.getNickname())
        users.add(user);
        return user;
    }
```

`CreateUserRequest`类
```
package agent.new_ai.ai.chat.usercruddemo.entity;

public class CreateUserRequest {
    String username;
    String nickname;
    int age;
    public CreateUserRequest(){

    }
    public CreateUserRequest(String username, String nickname, int age) {
        this.username = username;
        this.nickname = nickname;
        this.age = age;
    }
    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public String getNickname() {
        return nickname;
    }
    public void setNickname(String nickname) {
        this.nickname = nickname;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```
Controller层的`createUser`函数
```
@PostMapping("/users")
    public Result createUser(@RequestBody CreateUserRequest createUserRequest){
        return Result.*success*(userService.addUser(createUserRequest));
    }
```

## MayBatis实现

对于一个已有数据库的表对应使用MyBatis实现增删改查相应操作的一些步骤：
- 在SpringBoot项目中连接数据库
- 在pom文件中引入mysql和mybatis相关依赖
- 在yml文件或properties文件中写入数据库和mybatis的相关配置
- 实现和对应表各个字段的类，要求在数据库中使用_连接的字段需要使用驼峰形式实现例如数据库中字段为`user_id`在类中要实现为`userId`
- 在mapper层实现对应表中的增删改查操作具体实现逻辑与业务查询相关
- 然后在需要调用增删改查的Service层就可以调用了这样就实现了利用mybatis实现对数据库的操作

### 相关代码示例
`pom`文件，需要注意的是，引入的依赖必须是当前maven版本和java版本所支持的例如java17以上+springboot3.x需要mybatis3.x版本
```
 <!-- MyBatis Spring Boot Starter -->
        <!-- Source: https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>3.0.3</version>
        </dependency>
        <!-- MySQL数据库驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
```

`yml`配置文件
```
spring:
  datasource:
    url: jdbc:mysql://{自己的数据库地址}/{数据库名称}?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: {用户名}
    password: {用户密码}
    driver-class-name: com.mysql.cj.jdbc.Driver
```
>实现增删改查基本操作这么配置即可
```
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```
`mapper`层代码函数返回值根据Service层想要实现的相应功能返回
```
@Mapper
public interface UserMapper {

    @Select("select * from users where id = #{id}")
    User getUserById(@Param("id") int id);

    @Insert("insert into users(username,nickname,age,created_at) values (#{username},#{nickname},#{age},#{createdAt})")
    int insertUser(User user);

    @Update("update users set nickname = #{nickname} where id = #{id}")
    int updateUser(User user);

    @Delete("delete from users where id = #{id}")
    int deleteUser(@Param("id") int id);

    @Select("select * from users")
    List<User> getUsers();
}
```
