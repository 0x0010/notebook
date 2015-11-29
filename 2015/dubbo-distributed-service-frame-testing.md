#RPC框架Dubbo可行性验证
DUBBO是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，是阿里巴巴SOA服务化治理方案的核心框架，每天为2,000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。更多关于dubbo的介绍可以参考Dubbo主页[Dubbo.io](http://dubbo.io/)
##服务部署策略
当前我们的架构是应用和服务部署在一个JVM中，并做逻辑解耦。如下图：
![当前系统的技术架构图](http://7xlj4k.com1.z0.glb.clouddn.com/system-arch-1.png "当前系统架构图")
采用当前这样的架构对于我们现阶段的业务支持没有任何问题，并且从编码和部署都非常便捷，毕竟我们现在还处于系统快速迭代的时期。但是这些『优点』并不能完全说服我们坚持当前的架构。这种架构的蹩脚之处也是显而易见的，最大的缺点就是无法保证服务的高可用性。考虑到这一点如何保证我们的服务高可用？
##如何实现服务高可用
考虑到这一点，并借助于Dubbo，可以将我们的服务独立出来进行物理分离。再不考虑其他因素（比如DB）的影响，只需要两个服务进程就可以实现我们服务的高可用，避免出现系统更新造成服务不可用的风险。下图是我设想的系统结构图：
![理想的系统技术架构图](http://7xlj4k.com1.z0.glb.clouddn.com/system-arch-2.png "理想的系统架构图")
* Service Provider是服务的提供方，可以提供完整的服务。也就是说Service-Provider-Group中的每个Provider都可以提供完整的简职服务。
* Service Consumer不需要知道服务提供方的具体实现和部署形式，采用RPC远程调用服务。部署时，只需要将服务提供方的接口清单提供给调用方即可。

## Dubbo的集成与测试
###搭建Zookeeper（服务注册中心）
下载安装包zookeeper-3.4.6, 解压之后，拷贝一个zoo.cfg到conf目录，其他的暂时不修改，保留默认的2181端口。我虚拟的Linux机器的IP是192.168.37.128，所以Zookeeper服务的地址是zookeeper://192.168.37.128:2181
###部署两个服务进程（Service-Provider-Instance）
Provider包里的服务描述文件 serve-provider.xml
````
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <bean id="customerService" class="com.jianzhi.demo.provider.CustomerServiceImpl" />
    <dubbo:service interface="com.jianzhi.demo.CustomerService" ref="customerService" />
</beans>
````
网上很多教程是把服务暴露的配置卸载包的描述文件，我觉得这样不是很合理。包里不应该包含部署的信息。所以关于服务注册的信息，放在dubbo.properties文件中。
````
dubbo.container=log4j,spring
dubbo.application.name=serve-provider
dubbo.application.owner=william
#dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper://192.168.37.128:2181
#dubbo.registry.address=redis://127.0.0.1:6379
#dubbo.registry.address=dubbo://127.0.0.1:9090
#dubbo.monitor.protocol=registry
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880/20881
dubbo.service.loadbalance=roundrobin
#dubbo.log4j.file=logs/dubbo-demo-consumer.log
#dubbo.log4j.level=WARN
````
先启动一个服务提供者，执行com.alibaba.dubbo.container.Main即可。看到日志，提示服务注册成功。
````
// 部分日志
[30/11/15 12:24:41:041 CST] main  INFO zookeeper.ZooKeeper: Initiating client connection, connectString=192.168.37.128:2181 sessionTimeout=30000 watcher=org.I0Itec.zkclient.ZkClient@363a52f
[30/11/15 12:24:41:041 CST] main-SendThread()  INFO zookeeper.ClientCnxn: Opening socket connection to server /192.168.37.128:2181
[30/11/15 12:24:41:041 CST] main-SendThread(192.168.37.128:2181)  INFO zookeeper.ClientCnxn: Socket connection established to 192.168.37.128/192.168.37.128:2181, initiating session
[30/11/15 12:24:41:041 CST] main-SendThread(192.168.37.128:2181)  INFO zookeeper.ClientCnxn: Session establishment complete on server 192.168.37.128/192.168.37.128:2181, sessionid = 0x1515349c035000d, negotiated timeout = 30000
[30/11/15 12:24:41:041 CST] main-EventThread  INFO zkclient.ZkClient: zookeeper state changed (SyncConnected)
[30/11/15 12:24:41:041 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Register: dubbo://192.168.37.1:20880/com.jianzhi.demo.CustomerService?anyhost=true&application=serve-provider&dubbo=2.4.10&interface=com.jianzhi.demo.CustomerService&loadbalance=roundrobin&methods=getCustomerById&owner=william&pid=1325&side=provider&timestamp=1448814280809, dubbo version: 2.4.10, current host: 127.0.0.1
[30/11/15 12:24:41:041 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Subscribe: provider://192.168.37.1:20880/com.jianzhi.demo.CustomerService?anyhost=true&application=serve-provider&category=configurators&check=false&dubbo=2.4.10&interface=com.jianzhi.demo.CustomerService&loadbalance=roundrobin&methods=getCustomerById&owner=william&pid=1325&side=provider&timestamp=1448814280809, dubbo version: 2.4.10, current host: 127.0.0.1
[30/11/15 12:24:41:041 CST] main  INFO zookeeper.ZookeeperRegistry:  [DUBBO] Notify urls for subscribe url provider://192.168.37.1:20880/com.jianzhi.demo.CustomerService?anyhost=true&application=serve-provider&category=configurators&check=false&dubbo=2.4.10&interface=com.jianzhi.demo.CustomerService&loadbalance=roundrobin&methods=getCustomerById&owner=william&pid=1325&side=provider&timestamp=1448814280809, urls: [empty://192.168.37.1:20880/com.jianzhi.demo.CustomerService?anyhost=true&application=serve-provider&category=configurators&check=false&dubbo=2.4.10&interface=com.jianzhi.demo.CustomerService&loadbalance=roundrobin&methods=getCustomerById&owner=william&pid=1325&side=provider&timestamp=1448814280809], dubbo version: 2.4.10, current host: 127.0.0.1
[30/11/15 12:24:41:041 CST] main  INFO container.Main:  [DUBBO] Dubbo SpringContainer started!, dubbo version: 2.4.10, current host: 127.0.0.1
[2015-11-30 00:24:41] Dubbo service server started!
````
然后再启动另外一个服务提供者。 最终会有两个不同断口（20880和20881）的服务注册到zookeeper。
###调用测试
消费端配置，消费端不需要配置dubbo的信息，只需指定zookeeper地址即可。
````
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd
        ">

    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="serve_consumer" />

    <!-- 使用zookeeper注册中心暴露服务地址 -->
     <!--<dubbo:registry address="multicast://224.5.6.7:1234" />-->
    <dubbo:registry address="zookeeper://192.168.37.128:2181" />
    <!--<dubbo:registry -->

    <!-- 生成远程服务代理，可以像使用本地bean一样使用demoService -->
    <dubbo:reference id="customerService" interface="com.jianzhi.demo.CustomerService" />

</beans>
````
写一个简单的程序测试
````
    public static void main(String[] args) throws Exception{
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
        context.start();
        for(int i=0; i< 20; i++) {
            CustomerService customerService = (CustomerService)context.getBean("customerService");
            String remoteReturn = customerService.getCustomerById(10971L);
            System.out.println(remoteReturn);
            Thread.sleep(1000);
        }
    }
````

客户端日志
````
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
Hello 10971, response form provider: 192.168.37.1:20880
Hello 10971, response form provider: 192.168.37.1:20881
````

这就是我们想要的结果。
