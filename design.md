
# 基本原理

QT4A通过向被测Android应用进程中注入测试桩，以获取进程中的控件树信息，以及相关的类、对象的属性和方法。

测试桩是使用Java语言开发的jar包（dex），它主要利用Java的反射功能，获取到进程中类和对象的实例。

测试桩中会创建一个Socket服务端，客户端连接后可以获取到控件的ID、坐标、文本、可见性等信息，通过这些信息，用户可以对控件进行查找、获取文本、设置文本、点击、滑动等操作。

除此之外，测试桩还提供了反射获取对象属性、调用函数等能力，这使得QT4A拥有了`超越UI测试`的能力。使用者可以利用这些功能来做更多的事情。

## 如何注入测试桩

注入过程主要利用了`droid_inject`和`libdexloader.so`这两个文件。`droid_inject`主要是使用了`ptrace`调试接口，将自己变成被测进程的调试进程，从而拥有了读写寄存器、读写内存等能力。

### 模拟函数调用的原理

1. 按照当前CPU架构的函数调用约定，构造好参数和堆栈
2. 修改`EIP寄存器`（x86）或`PC寄存器`（arm），跳转到目标函数地址执行
3. 从寄存器获取函数执行结果

### 注入SO模块的原理

1. 获取`dlopen`函数在目标进程中的地址
2. 在目标进程中调用`dlopen`函数加载SO模块
3. 获取SO模块的入口函数地址
4. 在目标进程中调用入口函数

### SO中加载测试桩的原理

1. 调用`libandroid_runtime.so`模块中的`getJNIEnv`函数获取当前线程的`JNIEnv*`指针
2. 根据`JNIEnv*`指针获取`JavaVM*`指针
3. 创建子线程，根据`JavaVM*`指针获取子线程的`JNIEnv*`指针（使用子线程可以提升成功率）
4. 获取`dalvik.system.DexClassLoader`类实例以及构造函数
5. 实例化`DexClassLoader`对象，传入要加载的`dex`路径
6. 获取`dex`中的入口函数，并执行

### 使用限制

`ptrace`接口只能在以下两种情况下使用：

* `root`设备上可以注入任意进程
* 非`root`设备上只能注入相同`uid`的进程

因此，对于非`root`设备，需要使用应用的`debug`包。由于部分设备的`run-as`命令存在`bug`，这种情况下需要应用进行重打包。

## 应用测试桩实现原理

### 数据转发方法

测试桩运行后会创建一个`LocalSocket`服务端，PC端要访问该服务可以使用以下两种方法：

* 使用`adb forward`命令将服务映射到本地的`TCP`服务
* 直接使用`adb`协议创建一个透传的`socket`通道

第一种方法实现简单，但是存在需要创建本地端口，可能会出现端口冲突问题，影响到性能和稳定性。

第二种方法需要实现`adb`协议，但是不需要创建本地端口，性能和稳定性都优于第一种方法。

### 通信协议

测试桩使用了`JSON`格式数据进行通信，理论上支持任意语言访问。主要包含以下字段：

* `Cmd` 命令字，当前执行的操作名
* `Seq` 序号
* `其它字段` 都是请求的参数

返回结果会包含`Result`字段，里面是命令执行的结果；执行报错时会包含`Error`字段。

### 对象映射方式

跨进程（跨设备）通信，需要解决的一个问题，就是如何将两个进程中的对象建立一对一的映射关系。

Java中每个对象都有一个内存地址相关的`Hashcode`值，QT4A中使用这个值作为对象的唯一标识，查找控件时返回的也是这个值。之后所有的控件操作都会传入这个值，测试桩会在控件树中根据这个值查找对应的控件实例。

### 测试桩基本思想

测试桩主要利用了Java的`反射机制`，可以在运行时获取到类、对象、属性、方法等实例，从而访问到整个`Android`世界。

因此，所有逻辑的入口只能是`类`以及`静态方法`或`静态变量`，这些属于可以直接反射获取的范畴。

总的来说，基本思想就是：`从不变到会变`，`由已知到未知`。

### 如何获取控件树

```java
public final class WindowManagerGlobal {
    private static WindowManagerGlobal sDefaultWindowManager;
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
}
```

WindowManagerGlobal这个类中包含了当前进程中所有控件树的根的列表`mRoots`，同时它的实现是个单例，`sDefaultWindowManager`中保存了该类的实例。因此，可以使用以下过程获取所有控件树。

```
+-------------------------------------------+
| WindowManagerGlobal.sDefaultWindowManager |
+------------------+------------------------+
                   |
                   |
                   |
                   |
                   v
     +-------------+--------------+
     | WindowManagerGlobal object |
     +-------------+--------------+
                   |
                   |                  +-------+            +-----------+
                   |     +----------> | mView | +--------> | mChildren |
                   |     |            +-------+            +-----------+
                   v     |
              +----+---+ |            +-------+            +-----------+
              | mRoots | +----------> | mView | +--------> | mChildren |
              +--------+ |            +-------+            +-----------+
                         |
                         |            +-------+            +-----------+
                         +----------> | mView | +--------> | mChildren |
                                      +-------+            +-----------+

```

## 系统测试桩实现原理

### 系统测试桩简介

在自动化测试中，除了要对应用进行操作，还需要支持对系统的操作，比如：WIFI、剪切板、截屏、屏幕解锁、权限控制等。这些有部分可以通过shell命令实现，但是大部分的功能都需要额外实现。系统测试桩主要就是用于对Android系统的控制。

### 实现方式

系统测试桩也是使用Java语言开发，以独立进程方式运行。主要以下两种执行方式：

* 命令行方式
* 服务进程方式

前者适合低频、返回结果简单的场景，这种方式需要每次都创建新的进程，执行时间会稍长一些；后者适合高频或返回结果复杂的场景，这种方式使用和应用测试桩相同的方式通信，耗时较短。

在非`root`手机上，有些操作使用`shell`权限是无法完成的（比如操作WIFI），此时，是使用`QT4A助手`创建的后台服务进程来操作设备。

## Python层实现分析

### 总体结构

Python层主要分为两层，底层是对应用测试桩和系统测试桩接口的封装，上层主要是QT4A对用户的接口。

底层主要包含`adb`、`androiddriver`、`devicedriver`、`webdriver`等模块。类关系如下：

```
    +-----+                   +------------+
    | ADB |                   | IWebDriver |
    +--+--+                   +------+-----+
       ^                             ^
       |                             |
       |                             |
+------+-------+             +-------+-------+
| DeviceDriver |             |WebkitWebDriver|
+------+-------+             +-------+-------+
       ^                             ^
       |                             |
       |                             |
+------+--------+              +-----+-----+
| AndroidDriver |              | WebDriver |
+---------------+              +-----------+

```

最底层的类是`ADB`，它主要提供了ADB相关的操作，以及常见的`shell`命令封装，所有对手机的操作最终都会转化为ADB操作。

DeviceDriver是对系统测试桩接口的封装，大部分都是使用了命令行方式，少数使用了服务进程方式。

AndroidDriver是对应用测试桩接口的封装。

WebDriver是针对Android端的`qt4w`中的`IWebDriver`接口实现，用于支持Android端的Web自动化测试。

上层主要包含`device`、`androidapp`、`andrcontrols`、`androidtestbase`等模块。各模块主要功能如下：

* `device` 对设备接口的进一步封装，用户主要使用这里的接口
* `androidapp` 应用基类，所有项目都需要创建一个`AndroidApp`类的子类作为自己的应用类
* `andrcontrols` 窗口基类和Android常用的控件封装，包括`TextView`、`EditText`、`Button`、`ImageView`、`ScrollView`、`ListView`等
* `androidtestbase` 测试基类，用户需要创建一个`AndroidTestBase`的子类作为自己的测试基类


