## Java 中的类何时被加载器加载 
在 Java 程序启动的时候，并不会一次性加载程序中所有的 .class 文件，而是在程序的运行过程中，动态地加载相应的类到内存中。

通常情况下,Java 程序中的 .class 文件会在以下 2 种情况下被 ClassLoader 主动加载到内存中：

1. 调用类构造器
2. 调用类中的静态（static）变量或者静态方法

## Java 中 ClassLoader

JVM 中自带 3 个类加载器：
1. 启动类加载器 BootstrapClassLoader
2. 扩展类加载器 ExtClassLoader （JDK 1.9 之后，改名为 PlatformClassLoader） 
3. 系统加载器 APPClassLoader

以上 3 者在 JVM 中有各自分工，但是又互相有依赖。

## APPClassLoader 系统类加载器

部分源码如下：

[![APPClassLoader 系统类加载器.png](https://z3.ax1x.com/2021/08/01/Wzl5h6.png)](https://imgtu.com/i/Wzl5h6)

可以看出，AppClassLoader 主要加载系统属性“java.class.path”配置下类文件，也就是环境变量 CLASS_PATH 配置的路径。因此 AppClassLoader 是面向用户的类加载器，我们自己编写的代码以及使用的第三方 jar 包通常都是由它来加载的。

## ExtClassLoader 扩展类加载器

部分源码如下：

[![ExtClassLoade 扩展类加载器.png](https://z3.ax1x.com/2021/08/01/Wz1iuQ.png)](https://imgtu.com/i/Wz1iuQ)

可以看出，ExtClassLoader 加载系统属性“java.ext.dirs”配置下类文件，可以打印出这个属性来查看具体有哪些文件：

[![ExtClassLoade 扩展类加载器02.png](https://z3.ax1x.com/2021/08/01/Wz1mCV.png)](https://imgtu.com/i/Wz1mCV)

结果如下：

[![ExtClassLoade 扩展类加载器03.png](https://z3.ax1x.com/2021/08/01/Wz1vqJ.png)](https://imgtu.com/i/Wz1vqJ)

**ExtClassLoade 扩展类加载器 主要负责加载Java的扩展类库,默认加载JAVA_HOME/jre/lib/ext/目录下的所有jar包或者由java.ext.dirs系统属性指定的jar包.放入这个目录下的jar包对AppClassLoader加载器都是可见的(因为ExtClassLoader是AppClassLoader的父加载器,并且Java类加载器采用了委托机制)。**

## BootstrapClassLoader 启动类加载器

BootstrapClassLoader 同上面的两种 ClassLoader 不太一样。

首先，它并不是使用 Java 代码实现的，而是由 C/C++ 语言编写的，它本身属于虚拟机的一部分。因此我们无法在 Java 代码中直接获取它的引用。如果尝试在 Java 层获取 BootstrapClassLoader 的引用，系统会返回 null。

BootstrapClassLoader 加载系统属性“sun.boot.class.path”配置下类文件，可以打印出这个属性来查看具体有哪些文件：

[![BootstrapClassLoader 启动类加载器.png](https://z3.ax1x.com/2021/08/01/Wz8a0H.png)](https://imgtu.com/i/Wz8a0H)

结果如下：

[![BootstrapClassLoader 启动类加载器1.png](https://z3.ax1x.com/2021/08/01/Wz8W7j.png)](https://imgtu.com/i/Wz8W7j)

可以看到，这些全是 JRE 目录下的 jar 包或者 .class 文件。

## 双亲委派模式（Parents Delegation Model）

既然 JVM 中已经有了这 3 种 ClassLoader，那么 JVM 又是如何知道该使用哪一个类加载器去加载相应的类呢？答案就是：**双亲委派模式**。

**双亲委派模式**

所谓双亲委派模式就是，当类加载器收到加载类或资源的请求时，通常都是先委托给父类加载器加载，也就是说，只有当父类加载器找不到指定类或资源时，自身才会执行实际的类加载过程。

其具体实现代码是在 ClassLoader.java 中的 loadClass 方法中，如下所示：

[![双亲委派模式1.png](https://z3.ax1x.com/2021/08/01/WzG9gK.png)](https://imgtu.com/i/WzG9gK)

解释说明：

1. 判断该 Class 是否已加载，如果已加载，则直接将该 Class 返回。
2. 如果该 Class 没有被加载过，则判断 parent 是否为空，如果不为空则将加载的任务委托给parent。
3. 如果 parent == null，则直接调用 BootstrapClassLoader 加载该类。
4. 如果 parent 或者 BootstrapClassLoader 都没有加载成功，则调用当前 ClassLoader 的 findClass 方法继续尝试加载。

那这个 parent 是什么呢？ 我们可以看下 ClassLoader 的构造器，如下：

[![双亲委派模式2.png](https://z3.ax1x.com/2021/08/01/WzG226.png)](https://imgtu.com/i/WzG226)

## 举例说明

比如执行以下代码：

	Test test = new Test();

默认情况下，JVM 首先使用 AppClassLoader 去加载 Test 类。

1. AppClassLoader 将加载的任务委派给它的父类加载器（parent）—ExtClassLoader。
2. ExtClassLoader 的 parent 为 null，所以直接将加载任务委派给 BootstrapClassLoader。
3. BootstrapClassLoader 在 jdk/lib 目录下无法找到 Test 类，因此返回的 Class 为 null。
4. 因为 parent 和 BootstrapClassLoader 都没有成功加载 Test 类，所以AppClassLoader会调用自身的 findClass 方法来加载 Test。

最终 Test 类就是被 AppClassLoader 加载到内存中，可以通过如下代码印证此结果：

[![双亲委派模式3.png](https://z3.ax1x.com/2021/08/01/WzGvqg.png)](https://imgtu.com/i/WzGvqg)

打印结果为：

[![双亲委派模式4.png](https://z3.ax1x.com/2021/08/01/WzJEsU.png)](https://imgtu.com/i/WzJEsU)

可以看出，Test 的 ClassLoader 为 AppClassLoader 类型，而 AppClassLoader 的 parent 为 ExtClassLoader 类型。ExtClassLoader 的 parent 为 null。

**注意：**“双亲委派”机制只是 Java 推荐的机制，并不是强制的机制。我们可以继承 java.lang.ClassLoader 类，实现自己的类加载器。如果想保持双亲委派模型，就应该重写 findClass(name) 方法；如果想破坏双亲委派模型，可以重写 loadClass(name) 方法。

## 自定义 ClassLoader

JVM 中预置的 3 种 ClassLoader 只能加载特定目录下的 .class 文件，如果我们想加载其他特殊位置下的 jar 包或类时（比如，我要加载网络或者磁盘上的一个 .class 文件），默认的 ClassLoader 就不能满足我们的需求了，所以需要定义自己的 Classloader 来加载特定目录下的 .class 文件。

## 自定义 ClassLoader 步骤
1. 自定义一个类继承抽象类 ClassLoader。
2. 重写 findClass 方法。
3. 在 findClass 中，调用 defineClass 方法将字节码转换成 Class 对象，并返回。

用一段伪代码来描述这段过程如下：

[![自定义 ClassLoader 步骤.png](https://z3.ax1x.com/2021/08/01/WzYC0e.png)](https://imgtu.com/i/WzYC0e)

## 自定义 ClassLoader 实践

首先在本地电脑上创建一个测试类 Secret.java，代码如下：

[![自定义 ClassLoader 实践1.png](https://z3.ax1x.com/2021/08/01/WzYAfI.png)](https://imgtu.com/i/WzYAfI)


测试类所在磁盘路径如下图：

[![自定义 ClassLoader 实践2.png](https://z3.ax1x.com/2021/08/01/WzYUnU.png)](https://imgtu.com/i/WzYUnU)

接下来，创建 DiskClassLoader 继承 ClassLoader，重写 findClass 方法，并在其中调用 defineClass 创建 Class，代码如下：

[![自定义 ClassLoader 实践3.png](https://z3.ax1x.com/2021/08/01/WztZ59.png)](https://imgtu.com/i/WztZ59)

最后，写一个测试自定义 DiskClassLoader 的测试类，用来验证我们自定义的 DiskClassLoader 是否能正常 work。

[![自定义 ClassLoader 实践4.png](https://z3.ax1x.com/2021/08/01/WztlDO.png)](https://imgtu.com/i/WztlDO)

解释说明：

+ ①代表需要动态加载的 class 的路径。
+ ②代表需要动态加载的类名。
+ ③代表需要动态调用的方法名称。

最后执行上述 testClassLoader 方法，并打印如下结果，说明我们自定义的 DiskClassLoader 可以正常工作。

[![自定义 ClassLoader 实践5.png](https://z3.ax1x.com/2021/08/01/WztvqO.png)](https://imgtu.com/i/WztvqO)

**注意：**上述动态加载 .class 文件的思路，经常被用作热修复和插件化开发的框架中，包括 QQ 空间热修复方案、微信 Tink 等原理都是由此而来。客户端只要从服务端下载一个加密的 .class 文件，然后在本地通过事先定义好的加密方式进行解密，最后再使用自定义 ClassLoader 动态加载解密后的 .class 文件，并动态调用相应的方法。

## Android 中的 ClassLoader

本质上，Android 和传统的 JVM 是一样的，也需要通过 ClassLoader 将目标类加载到内存，类加载器之间也符合双亲委派模型。但是在 Android 中， ClassLoader 的加载细节有略微的差别。

在 Android 虚拟机里是无法直接运行 .class 文件的，Android 会将所有的 .class 文件转换成一个 .dex 文件，并且 Android 将加载 .dex 文件的实现封装在 BaseDexClassLoader 中，而我们一般只使用它的两个子类：PathClassLoader 和 DexClassLoader。

## PathClassLoader

PathClassLoader 用来加载系统 apk 和被安装到手机中的 apk 内的 dex 文件。它的 2 个构造函数如下：

[![PathClassLoader_1.png](https://z3.ax1x.com/2021/08/01/WzNlzq.png)](https://imgtu.com/i/WzNlzq)

参数说明：

+ dexPath：dex 文件路径，或者包含 dex 文件的 jar 包路径；
+ librarySearchPath：C/C++ native 库的路径。

PathClassLoader 里面除了这 2 个构造方法以外就没有其他的代码了，具体的实现都是在 BaseDexClassLoader 里面，其 dexPath 比较受限制，一般是已经安装应用的 apk 文件路径。

当一个 App 被安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的，可以通过如下代码验证：

[![PathClassLoader_2.png](https://z3.ax1x.com/2021/08/01/WzNBS1.png)](https://imgtu.com/i/WzNBS1)

打印结果如下：

[![PathClassLoader_3.png](https://z3.ax1x.com/2021/08/01/WzU3hd.png)](https://imgtu.com/i/WzU3hd)

## DexClassLoader

先来看官方对 DexClassLoader 的描述：

	A class loader that loads classes from .jar and .apk filescontaining a classes.dex entry. 

	This can be used to execute code notinstalled as part of an application.

很明显，对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的 dex 的加载。

DexClassLoader 的源码里面只有一个构造方法，代码如下：

[![DexClassLoader_1.png](https://z3.ax1x.com/2021/08/01/WzUy3n.png)](https://imgtu.com/i/WzUy3n)

参数说明：

+ **dexPath：**包含 class.dex 的 apk、jar 文件路径 ，多个路径用文件分隔符（默认是“:”）分隔。
+ **optimizedDirectory：**用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径。

## 使用 DexClassLoader 实现热修复

理论知识都是为实践作基础，接下来我们就使用 DexClassLoader 来模拟热修复功能的实现。

### 创建 Android 项目 DexClassLoaderHotFix

项目结构如下：

[![DexClassLoader_1.png](https://z3.ax1x.com/2021/08/01/WzarVO.png)](https://imgtu.com/i/WzarVO)

ISay.java 是一个接口，内部只定义了一个方法 saySomething。

[![DexClassLoader_2.png](https://z3.ax1x.com/2021/08/01/WzayIe.png)](https://imgtu.com/i/WzayIe)

SayException.java 实现了 ISay 接口，但是在 saySomething 方法中，打印“something wrong here”来模拟一个线上的 bug。

[![DexClassLoader_3.png](https://z3.ax1x.com/2021/08/01/WzaRxI.png)](https://imgtu.com/i/WzaRxI)

最后在 MainActivity.java 中，当点击 Button 的时候，将 saySomething 返回的内容通过 Toast 显示在屏幕上。

[![DexClassLoader_4.png](https://z3.ax1x.com/2021/08/01/WzaHiQ.png)](https://imgtu.com/i/WzaHiQ)

最后运行效果如下：

[![DexClassLoader_5.png](https://z3.ax1x.com/2021/08/01/WzaLzn.png)](https://imgtu.com/i/WzaLzn)

### 创建 HotFix patch 包

新建 Java 项目，并分别创建两个文件 ISay.java 和 SayHotFix.java。

[![创建 HotFix patch 包01.png](https://z3.ax1x.com/2021/08/01/WzdFzR.png)](https://imgtu.com/i/WzdFzR)

[![创建 HotFix patch 包02.png](https://z3.ax1x.com/2021/08/01/WzdeeK.png)](https://imgtu.com/i/WzdeeK)

ISay 接口的包名和类名必须和 Android 项目中保持一致。SayHotFix 实现 ISay 接口，并在 saySomething 中返回了新的结果，用来模拟 bug 修复后的结果。

将 ISay.java 和 SayHotFix.java 打包成 **say_something.jar**，然后通过 dx 工具将生成的 **say_something.jar** 包中的 class 文件优化为 dex 文件。

	dx --dex --output=say_something_hotfix.jar say_something.jar

上述 **say_something_hotfix.jar** 就是我们最终需要用作 hotfix 的 jar 包。

**将 HotFix patch 包拷贝到 SD 卡主目录，并使用 DexClassLoader 加载 SD 卡中的 ISay 接口**

	adb push say_something_hotfix.jar /storage/self/primary/ 

接下来，修改 MainActivity 中的逻辑，使用 DexClassLoader 加载 HotFix patch 中的 SayHotFix 类，如下：

[![创建 HotFix patch 包03.png](https://z3.ax1x.com/2021/08/01/Wzd6mV.png)](https://imgtu.com/i/Wzd6mV)

**注意：**因为需要访问 SD 卡中的文件，所以需要在 AndroidManifest.xml 中申请权限

最后运行效果如下：

[![创建 HotFix patch 包04.png](https://z3.ax1x.com/2021/08/01/WzdRkF.png)](https://imgtu.com/i/WzdRkF)

## 总结

+ **ClassLoader 就是用来加载 class 文件的，不管是 jar 中还是 dex 中的 class。**
+ **Java 中的 ClassLoader 通过双亲委托来加载各自指定路径下的 class 文件。**
+ **可以自定义 ClassLoader，一般覆盖 findClass() 方法，不建议重写 loadClass 方法。**
+ **Android 中常用的两种 ClassLoader 分别为：PathClassLoader 和 DexClassLoader。**
