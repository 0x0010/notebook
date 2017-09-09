光连接池的并发包（ConcurrentBag）中使用了ThreadLocal来避免并发竞争，从而提升并发性能。今天想针对这个ThreadLocal做个测试，看看使用这个线程变量究竟给性能带来多大提升。

这边文章主要测试两点：
- 连接池在使用和不使用ThreadLocal的情况下，性能有多大差别
- 使用ThreadLocal的情况下，性能与并发数的变化曲线

测试场景准备：使用Springboot搭建一个单表查询接口。并发场景使用JMeter模拟。

### 使用ThreadLocal时的性能曲线
这个测试无需对源码做任何修改，直接使用App测试即可。
