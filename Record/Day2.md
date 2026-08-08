## User、UserService、Main、BusinessException 的职责
> User用来描述用户的哪些属性和行为，UserService是业务逻辑层，主要是写添加用户、查询删除修改等业务规则，Main是程序的入口，负责的是读取用户输入和调用UserService里面的一些方法、输出结果。BusinessException主要是当业务规则出现错误时进行异常抛出，证明程序没有坏，是当前操作不符合业务规则

## Controller 和 Service 的区别
> `Controller`负责的是接收HTTP请求和请求参数，主要是调用`Service`层的方法，将结果返回给前端，`Service`负责业务处理，比如说CRUD或者其他业务操作

## Entity 和 DTO 的区别
> `Entity` 通常表示和数据库表结构对应的数据对象，比如 `UserEntity` 对应用户表。`DTO` 是数据传输对象，通常用于接口请求或响应，比如新增用户请求可以用 `CreateUserRequest`，返回用户信息可以用 `UserResponse`。

## 控制台输入输出迁移到Web会变成什么
> 输入会变成HTTP请求参数、路径参数或JSON请求体，输出会变成HTTP响应，返回JSON数据

## final的含义
> `final`限制的变量是不能被重新赋值，赋值后就无法指向其它的对象

## 前端传来请求的过程
> 对于前端传来的HTTP请求Controller层接收该请求后调用相应的Service的方法然后根据方法的内容，可以创建一个Result类将结果返回为统一格式，可以创建全局异常处理器来统一的返回异常信息给前端
在统一返回的结果中可以自定义返回方式，一般的返回都是JSON数据可能存储状态码，响应提示信息和返回数据信息等
对于统一返回的类也可以定义一些成功或失败的静态方法，这样就不用每次手动手写返回信息

## 为什么后端不能相信前端传来的 User 数据一定是合法的
> 因为前端的校验有可能会被骗过去，请求可能会来自脚本等恶意调用，所以必须确认数据合法性

## 对于用户名称是否为空的判断
- user.getName() == "" 不推荐。Java 中 == 比较的是字符串对象地址，不是字符串内容。判断空字符串应该用 .isEmpty() 或 .equals("")。
- user.getName() == null 必须放在 user.getName().isEmpty() 前面，否则如果 name 为 null，先调用方法会出现空指针异常。

# 泛型、Lambda、Stream

## 泛型
ArrayList<User> 里面的User就是泛型，表示这个集合中只能存放User类型的数据，不需要自己强制类型转换

## Stream
这里的Stream并不是大模型回答时的"流式输出"，这是时集合数据处理管道操作，用来对集合筛选，转换，排序等操作
Stream的筛选方式
> users.stream()
    .filter(user -> user.getAge() >= 18)
    .toList();

## 对于创建的User对象的输出
> System.out.println(user) 会自动调用 user.toString()。
如果 User 类没有重写 toString()，就会调用 Object 默认的 toString()。
Object 默认的 toString() 输出的是 类名@哈希值，不是对象属性值。
所以说如果想要输出User对象的属性值，就可以对User类中的toString方法进行重写类似以下代码
```
@Override
public String toString() {
    return "User{" +
            "id=" + id +
            ", name='" + name + '\'' +
            ", age=" + age +
            '}';
}
```
