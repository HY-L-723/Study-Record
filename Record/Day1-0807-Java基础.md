# DAY1
## 类与对象笔记

1.类和对象的区别

类是描述一类事物的模板，定义了属性和方法；对象是根据类创建出来的具体实例，每个对象都有自己独立的属性值，并可以调用类中定义的方法

2.成员变量和局部变量的区别

成员变量定义在类中、方法外，属于对象；局部变量定义在方法或代码块中，只能在当前方法或代码块内使用。成员变量有默认值，局部变量必须手动赋值后才能使用。

3.构造方法有什么用

构造方法是在创建对象时自动调用的方法，主要作用是初始化对象的成员变量，让对象一创建出来就具有必要的数据。控制属性不被随意修改主要依赖 private 封装和 setter 校验。

4.this关键字代表什么

this 表示当前对象的引用。常用于区分成员变量和局部变量，也可以用来调用当前对象的其他方法或构造方法。

5.UserService层是干什么的

UserService 是业务逻辑层，负责处理用户相关操作，比如新增用户、查询用户、删除用户、修改昵称、判断用户名是否重复。Main 只负责接收输入和调用 UserService，不直接处理复杂业务。

## Java 集合笔记

### ArrayList

ArrayList 底层是数组，特点是容量可以自动扩容。

优点：

- 根据下标查询快，时间复杂度 O(1)
- 使用方便，不需要手动维护 count
- 删除元素时会自动移动后面的元素

缺点：

- 在中间插入或删除元素时，需要移动后面的元素，时间复杂度 O(n)

适合场景：

- 按顺序存储数据
- 经常遍历
- 经常根据下标查询
- 插入删除不特别频繁

### HashMap

HashMap 是 key-value 结构。

例如：

`HashMap<Integer, User>`

其中：

- `Integer` 是 key，表示用户 id
- `User` 是 value，表示用户对象

常用方法：

- `put(key, value)`：添加或覆盖数据
- `get(key)`：根据 key 获取 value
- `remove(key)`：根据 key 删除
- `containsKey(key)`：判断 key 是否存在
- `values()`：获取所有 value

优点：

- 根据 key 查询很快，平均时间复杂度 O(1)

适合场景：

- 经常根据 id、用户名、订单号等唯一标识查询数据

### ArrayList 和 HashMap 的选择

如果主要是按顺序展示、遍历数据，用 ArrayList。

如果主要是根据唯一标识快速查询、删除、修改数据，用 HashMap。

### HashSet + equals/hashCode + HashMap 底层原理

对于以下代码
```
HashSet<User> users = new HashSet<>();

User u1 = new User(1, "张三");
User u2 = new User(1, "张三");

users.add(u1);
users.add(u2);

System.out.println(users.size());
```
的执行结果为2，是因为此时对于User类并没有重写HashCode和equals方法HashSet底层是根据HashMap实现hashCode() 相同并且 equals() 返回 true才会认为两者相同去重
因此要重写equals和hashCode代码如下：
```
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;

    User user = (User) o;
    return id == user.id && username.equals(user.username);
}

@Override
public int hashCode() {
    return Objects.hash(id, username);
}
```
HashSet 去重流程：
hashCode 不同：直接认为不同，放入
hashCode 相同 + equals 为 false：发生冲突，但不是重复，放入
hashCode 相同 + equals 为 true：认为重复，不放入
所以HashCode值相同的也有可能会保留，这种情况就是Hash冲突
- HashSet 去重依赖 hashCode + equals
- equals 相等则 hashCode 必须相等
- hashCode 相等但 equals 不等，属于哈希冲突
- HashMap 根据 hash 定位桶，所以查询通常很快
- 冲突严重时，链表可能转红黑树
- 数组较小时优先扩容，不急着树化
### == 和 equals的区别
==：
- 基本数据类型比较值
- 引用数据类型比较地址

equals()：
- 默认比较地址
- 如果类重写了 equals()，可以按照自定义规则比较内容
在比较对象时 == 只比较地址，equals默认比较的是地址，但可以重写比较内容

## LinkedList + 集合对比 + static 关键字

### 集合对比
- ArrayList：适合有序、可重复、按下标查询
- LinkedList：适合频繁操作头尾元素
- HashSet：适合去重
- HashMap：适合根据 key 快速查询 value

### ArrayList 和 LinkedList
- ArrayList 底层是动态数组，也可以理解为可自动扩容的数组。
- LinkedList 底层是双向链表，每个节点保存数据，同时保存前一个节点和后一个节点的引用。
- 查询区别：
    - ArrayList 根据下标查询快，因为数组可以通过下标直接计算元素位置，时间复杂度是 O(1)。
    - LinkedList 根据下标查询慢，因为链表不能通过下标直接计算地址，需要从头或尾一个节点一个节点找，时间复杂度是 O(n)。
- 插入删除区别：

  1.ArrayList：
    - 查询快
    - 中间插入/删除慢，因为要移动后面的元素

  2.LinkedList：
    - 按下标查询慢
    - 如果已经定位到节点，插入/删除只需要修改引用
    - 如果按下标插入/删除，仍然要先遍历定位
### static 关键字
static 修饰的成员属于类，不属于某一个具体对象。
这个类创建出来的所有对象，都会共享同一份 static 变量。

- 普通成员变量和static成员变量区别：

  普通成员变量：
    - 属于对象
    - 每创建一个对象，就会有一份自己的成员变量
    - 一个对象修改自己的成员变量，不会影响其他对象
    
  static 成员变量：
    - 属于类
    - 整个类只有一份
    - 所有对象共享这一份变量
    - 一个对象修改 static 变量，其他对象看到的也是修改后的值

### main方法为什么是static
- public：JVM 要能从外部访问这个方法
- static：不需要创建对象，可以直接通过类调用
- void：main 方法不需要返回值
- String[] args：接收命令行参数

程序启动时对象还没有创建。
JVM 需要直接通过类名找到并调用 main 方法。
所以 main 方法必须是 static。

## static 方法 + 非 static 方法 + Java 异常处理

static 方法：
- 属于类
- 可以通过 类名.方法名() 调用
- 不需要先创建对象

普通方法：
- 属于对象
- 必须先 new 出对象
- 再通过 对象名.方法名() 调用

#### static 方法为什么不能直接使用普通成员变量和 this

`static` 方法属于类，可以在没有创建对象的情况下直接调用。而普通成员变量属于对象，`this` 表示当前对象。如果对象还没有创建，`static` 方法就不知道要访问哪个对象的成员变量，也没有所谓的当前对象，所以不能直接使用普通成员变量和 `this`

错误示例：

```java
class Course {
    String name;
    static int count;

    public static void printCount() {
        System.out.println(count); // 可以，count 属于类
    }

    public static void printName() {
        System.out.println(name);      // 错误，name 属于对象
        System.out.println(this.name); // 错误，static 方法中没有 this
    }
}
```

如果 `static` 方法确实要访问普通成员变量，必须先拿到对象：

```java
public static void printName(Course course) {
    System.out.println(course.name);
}
```

#### 异常是什么

> 异常是程序运行过程中出现的不正常情况。比如用户输入格式错误、数组下标越界、空指针等。如果异常没有被处理，程序可能会直接中断。

#### try-catch 的作用

`try-catch` 可以捕获程序运行过程中可能出现的异常，避免程序因为异常直接中断。它不是让错误消失，而是让程序有机会进行提示、记录日志或执行补救逻辑

## 多 catch + finally

> 如果一段代码可能出现多种异常，可以使用多个 `catch` 分别捕获不同类型的异常。不同异常对应不同处理逻辑，这样提示更清楚，也更方便排查问题

代码示例
```
 try {
    int num = sc.nextInt();
    int[] arr = {1, 2, 3};
    System.out.println(arr[num]);
} catch (InputMismatchException e) {
    System.out.println("输入格式错误，请输入数字");
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("数组下标越界，请输入 0-2 之间的数字");
}
```

#### 多 catch 的顺序

多个 `catch` 同时存在时，应该先写子类异常，再写父类异常。因为异常匹配是从上往下执行的，如果先写父类 `Exception`，它会把很多子类异常都捕获掉，后面的具体异常就没有机会执行，甚至会直接编译报错。

代码示例
```
try {
     int age = sc.nextInt();
} catch (InputMismatchException e) {
    System.out.println("输入格式错误");
} catch (Exception e) {
   System.out.println("其他异常");
}
```
#### finally 的作用

> `finally` 里的代码通常会在 `try-catch` 结束后执行。不管 `try` 中有没有出现异常，也不管异常有没有被 `catch` 捕获，`finally` 通常都会执行

常见用途
```text
关闭资源
释放连接
关闭文件
关闭数据库连接
做收尾操作
```

对于重复逻辑的代码操作可以将对应部分封装成方法例如：

> 把读取整数和异常处理封装成 `readInt()` 方法，可以复用同一套输入校验逻辑，避免在每个 `nextInt()` 位置重复写 `try-catch`。这样代码更简洁、更容易维护，也能保证所有整数输入位置的异常处理规则一致

## throw + throws + 自定义业务异常

### 自定义业务异常

```java
package User;

public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}
```
```text
BusinessException 专门表示业务规则不满足导致的异常。
比如用户 id 已存在、用户不存在、参数不合法等。
```
UserService层
```java
public void addUser(User user) {
    if (existId(user.getId())) {
        throw new BusinessException("用户id已存在");
    }
    users.add(user);
}
```
Main主函数捕获
```java
try {
    userService.addUser(user);
} catch (BusinessException e) {
    System.out.println(e.getMessage());
}
```
```
新增用户时 id 重复：输出“用户id已存在”
删除不存在的用户：输出“用户不存在，删除失败”
修改不存在的用户：输出“用户不存在，修改失败”
查询不存在的用户：输出“用户不存在”
```

#### throw 的含义

> `throw` 写在方法内部，表示真正主动抛出一个异常对象。当方法发现当前条件不满足业务规则，自己又不能继续正常完成任务时，就可以主动 `throw` 一个异常，把问题交给调用者处理。

示例
```
public void addUser(User user) {
    if (user == null) {
        throw new RuntimeException("用户不能为空");
    }
}
```

#### throws 的含义
`throws` 写在方法声明上，表示这个方法可能会抛出某些异常，提醒调用者处理。

示例
```
public void readFile() throws IOException {
    // 可能出现 IOException
}
```

#### 系统异常和业务异常
```text
系统异常：
- 程序运行环境或底层操作出问题
- 比如空指针、数组越界、数据库连接失败、文件读取失败

业务异常：
- 用户操作或业务条件不满足规则
- 比如 id 重复、用户不存在、余额不足、库存不足、权限不足
```

#### 为什么使用 BusinessException
`RuntimeException` 太宽泛，只能说明程序运行时出现了异常，但看不出这是业务规则不满足。`BusinessException` 语义更清楚，一看就知道这是业务异常，比如用户 id 已存在、用户不存在、余额不足、权限不足等。

#### 和 Spring Boot 全局异常处理的关系

控制台写法
```text
Main 调 UserService
UserService 抛 BusinessException
Main catch BusinessException
Main 输出 e.getMessage()
```

SpringBoot中

```text
Controller 调 Service
Service 抛 BusinessException
GlobalExceptionHandler 捕获 BusinessException
统一返回错误响应
```

### switch 的使用场景

`switch` 更适合对同一个变量进行多个固定值匹配的场景，比如菜单编号、状态码、类型编号。`if-else` 更适合复杂条件判断，比如范围判断、多个条件组合判断

### 为什么删除ArrayList元素时推荐普通for

普通`for`可以通过下标明确删除指定位置的元素，并且删除后可以立刻`return`或调整下标。增强`for`本质上使用迭代器遍历，如果在遍历过程中直接调用`users.remove(user)`修改集合结构，容易出现并发修改异常。

### 成功提示应该放在哪里

> 成功提示更适合放在 `Main`。因为当前项目中 `Main` 负责控制台交互和输出提示，`UserService` 负责业务规则，比如新增、删除、修改是否允许执行。

### User、UserService、Main对应哪一层

`User` 是数据模型类，对应 Spring Boot 中的 `Entity` 或 `DTO`；`UserService` 对应 `Service` 层；`Main` 在当前控制台项目里相当于程序入口和交互控制层，思想上接近以后 Spring Boot 中的 `Controller`。

### 为什么业务失败适合抛BusinessException


> 业务失败抛 `BusinessException`，可以明确表达“程序没坏，是业务规则不满足”。比如 id 重复、用户不存在、权限不足，这些都不是系统崩溃，而是当前操作不能继续。调用者捕获 `BusinessException` 后，可以统一输出或返回清晰的错误信息。

