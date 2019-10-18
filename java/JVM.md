# JVM

## JAVA 内存模型
---

<img src="attachments/c3b70793.png" width="500"/>


**1、Java虚拟机栈：** 虚拟机调用执行Java方法的内存

局部变量表

操作数栈 —— JAVA虚拟机基于栈，Android虚拟机基于寄存器

动态链接 —— 指向符号引用，提供栈对象实例化后堆对应的类地址

方向返回地址

**2、方法区**：已加载的类信息和运行时常量池

类信息Class：类型信息，类型的常量池，字段信息，方法信息，类变量(static)

  指向类加载器的，指向引用class实例的引用，方法表

运行时常量池：字面量(string,基本数据类型，final常量)，符号引用(类型，域，方法)

常量池解析：将常量池中的符号引用转换为直接指向方法区的引用

**3、Java堆：** 

存储对象实例，新生代(Eden8/from1/to1)+老生代


## 垃圾回收
---

1、引用计数法

2、标记清除算法 —— 老生代

3、复制算法 —— 新生代

4、标记压缩算法


## 类的加载
---

1、加载——由类加载器，查找字节嘛，并创建一个class对象

2、验证——文件格式，元数据，字节码，符号引用

3、准备——创建静态域，为类变量分配内存和初始值(方法区)

4、解析——符号引用替换为直接引用

5、初始化——指向类构造器（使用对象时初始化）

6、使用卸载


## 类加载器

1、类加载的层级结构：启动类加载器，扩展类加载器，系统类加载器

2、双亲委派模式：通过组合复用

```
protected Class<?> loadClass(String name, boolean resolve)
      throws ClassNotFoundException
  {
      synchronized (getClassLoadingLock(name)) {
          // 先从缓存查找该class对象，找到就不用重新加载
          Class<?> c = findLoadedClass(name);
          if (c == null) {
              long t0 = System.nanoTime();
              try {
                  if (parent != null) {
                      //如果找不到，则委托给父类加载器去加载
                      c = parent.loadClass(name, false);
                  } else {
                  //如果没有父类，则委托给启动加载器去加载
                      c = findBootstrapClassOrNull(name);
                  }
              } catch (ClassNotFoundException e) {
                  
              }

              if (c == null) {
                  // 如果都没有找到，则通过自定义实现的findClass去查找并加载
                  c = findClass(name);
              }
          }
          if (resolve) {//是否需要在加载时进行解析
              resolveClass(c);
          }
          return c;
      }
  }
```

#### 3、Android类加载器

系统类加载器：bootClassLoader, PathClassLoader,DexClassLoader(继承自BaseDexClassLoader)

pathClassLoader：支持加载Dex和加载已安装的Apk

DexClassLoader：加载未安装的Apk/dex/，可以从sd卡安装

BaseDexClassLoader:创建PathDexList对象，Element[] dexElements保存dex文件集合

    热修复：
    
    通过修改dexElements中dex文件顺序完成热修复
    
    通过native方法hook替换
  



## 对象的创建
---

**1、对象创建流程：**

  常量池定位符号引用
  
  类加载检查
  
  分配内存
  
  初始化零值
  
  设置对象头
  
  执行Init方法
  
2、内存布局：对象头/实例数据/对齐填充数据

3、对象访问定位：

通过栈上的reference数据操作堆上对象

句柄/直接指针


