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
