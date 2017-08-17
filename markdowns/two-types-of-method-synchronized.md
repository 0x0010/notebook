方法锁指的是带有synchronized关键字的方法。因为Java方法又包括静态方法和非静态方法。定义在静态方法上synchronized关键字的暂叫静态方法锁，非静态方法上的叫普通方法锁。

网络上有不少解释这两种锁的文章，但大多都是总结性的结论。比如，**普通方法锁锁的是对象，静态方法锁锁的是class**，因为静态方法与实例无关，所以锁也与实例无关。下面我们来看看Java是如何处理这两种锁的。

先写一个简单的测试类：

````java
package com.oxooio.codesnaps;

/**
 * Method synchronized analysis
 * 
 * @author Sam
 * @since 3.1.0
 */
public class SynchronizeMethodSmali {

  public static void main(String[] args) {
  }

  public SynchronizeMethodSmali() {

  }

  /**
   * 普通方法一
   */
  public synchronized void methodOne() {
    System.out.println("method one");
  }

  /**
   * 普通方法二
   */
  public synchronized void methodTwo() {
    System.out.println("method two");
  }

  /**
   * 静态方法一
   */
  public synchronized static void staticMethodOne() {
    System.out.println("static method one");
  }

  /**
   * 静态方法二
   */
  public synchronized static void staticMethodTwo() {
    System.out.println("static method two");
  }
}
````

这个类定义了两个普通方法和两个静态方法。四个方法都加了synchronized关键字。

### 普通方法锁

将两个普通方法编译成DalvikVM的指令如下

````assembly
.method public declared-synchronized methodOne()V
    .registers 3
    .prologue
    .line 18
    monitor-enter p0 #获取p0锁 p0表示this对象
    :try_start_1
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v1, "method one"
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V
    :try_end_8
    .catchall {:try_start_1 .. :try_end_8} :catchall_a
    .line 19
    monitor-exit p0 #释放p0锁
    return-void
    .line 18
    :catchall_a
    move-exception v0
    monitor-exit p0
    throw v0
.end method

.method public declared-synchronized methodTwo()V
    .registers 3
    .prologue
    .line 22
    monitor-enter p0 #获取p0锁
    :try_start_1
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream
    const-string v1, "method two"
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V
    :try_end_8
    .catchall {:try_start_1 .. :try_end_8} :catchall_a
    .line 23
    monitor-exit p0 #释放p0锁
    return-void
    .line 22
    :catchall_a
    move-exception v0
    monitor-exit p0
    throw v0
.end method
````

注意dalvik-opcodes中有”获取“和”释放“标记的代码行。从代码里可以看到，两个方法锁对象都是p0实例，也就是Java中的this实例，当有两个线程**同时分别**访问同一个实例的methodOne和methodTwo时，由于竞争的是同一把实例锁出现线程阻塞的情况。所以我们才会有普通方法锁等同于锁实例的结论。

### 静态方法锁

按照同样的方法将静态方法编译成dalvikvm指令

````assembly
.method public static declared-synchronized staticMethodOne()V
    .registers 3
    .prologue
    .line 26
    const-class v1, Lcom/oxooio/codesnaps/SynchronizeMethodSmali;
    monitor-enter v1 #获取类对象锁
    :try_start_3
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v2, "static method one"
    invoke-virtual {v0, v2}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V
    :try_end_a
    .catchall {:try_start_3 .. :try_end_a} :catchall_c
    .line 27
    monitor-exit v1 #释放类对象锁
    return-void
    .line 26
    :catchall_c
    move-exception v0
    monitor-exit v1
    throw v0
.end method

.method public static declared-synchronized staticMethodTwo()V
    .registers 3
    .prologue
    .line 31
    const-class v1, Lcom/oxooio/codesnaps/SynchronizeMethodSmali;
    monitor-enter v1 #获取类对象锁
    :try_start_3
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v2, "static method two"
    invoke-virtual {v0, v2}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V
    :try_end_a
    .catchall {:try_start_3 .. :try_end_a} :catchall_c
    .line 32
    monitor-exit v1 #释放类对象锁
    return-void
    .line 31
    :catchall_c
    move-exception v0
    monitor-exit v1
    throw v0
.end method
````

关注添加注释的monitor-enter和monitor-exit，v1是Lcom/oxooio/codesnaps/SynchronizeMethodSmali，表示类对象（class object），与实例（instance）没有关系。

### 总结

- 普通方法上加synchronized关键字，锁的是实例，并发访问同一个实例的相同方法或者不同方法会发生阻塞。
- 静态方法加synchronized关键字锁的是Class Object，与实例无关，所有调用加了锁的静态方法都要竞争锁。