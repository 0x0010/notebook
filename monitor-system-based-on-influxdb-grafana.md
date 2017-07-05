---
title: 使用influxdb和grafana快速开发监控系统
date: 2017/7/5 9:39
---

使用过云产品（或者做过运维工作）的同学对监控肯定都不陌生。通常，云厂商在管理员工具里都会提供监控功能，比如玲琅满目的图表以及告警等。毕竟在一个优秀的系统生态里，监控是非常重要的组成部分。通过监控系统，我们可以更直观地了解当前系统运行的状态，便于及时地保障系统的可用性，比如主机性能的升级，对一些核心API进行防止崩溃的限流等。

说到系统崩溃，不禁让我想起一段难忘的经历，那种无助与挣扎很是煎熬，不提也罢。都说吃一堑长一智，做系统也需要未雨绸缪，跟兵书里带兵打仗的道理一样一样的，如果没有沙盘，没有实时军情汇报，等待他们的只有兵败。

<!— more —>

### 监控是什么

个人理解，监控指标为两大类，分别是系统级指标和业务应用级指标。

**系统级指标**具体包括哪些，先看看wikipedia对系统监控的描述。

> Software monitors occur more commonly, sometimes as a part of a widget engine. These monitoring systems are often used to keep track of system resources, such as CPU usage and frequency, or the amount of free RAM. They are also used to display items such as free space on one or more hard drives, the temperature of the CPU and other important components, and networking information including the system IP address and current rates of upload and download. Other possible displays may include the date and time, system uptime, computer name, username, hard drive S.M.A.R.T. data, fan speeds, and the voltages being provided by the power supply.

总结起来，包括CPU、内存、存储、网络IO等，这些都是系统运维需要关心的数据。

对于**业务应用级指标**，没有绝对的标准，功能各异的业务应用有各自关心的指标。比如API类应用需要关心三个方面，接口是否可用（Availability），实时的性能如何（Performance）以及接口的正确性（Correctness）。参考[API Monitoring: Up Is Not Enough](https://www.pagerduty.com/blog/runscope-api-monitoring/)。

监控，包含“监”和“控”。这篇文章的主题聚焦在“监”字上，我们实现一个具备完整监控功能的Demo，包括CPU，网络，API请求次数以及API被请求的速率。

先看看最终的监控图表截图

![](https://ws3.sinaimg.cn/large/006tNc79gy1fh9de00y9bj31kw0jy0yp.jpg)

### 技术选型

说了这么多，是时候切入主题了。

先是技术选型

- springboot，业务系统使用springboot开发一个Http RESTful API。
- telegraf，influxdb家的采集工具，数据采集就用它了。[点击传送](https://github.com/influxdata/telegraf)
- influxdb，时序数据库，用于存储采集器采集到指标数据。[点击传送](https://github.com/influxdata/influxdb)
- grafana，基于时序数据库图形化展示数据的工具。[点击传送](https://grafana.com/)

个人非常喜欢grafana的图形界面，图表动态刷新，黑白两种主题可选，尤其在大屏展示时选择黑色主题非常酷。

### 安装时序数据库influxdb

数据库下载地址：https://portal.influxdata.com/downloads#influxdb

选择与自己环境相关的安装方式进行安装。

安装完毕，用浏览器访问AdminUI的地址http://influxdb-ip-address:8083

![](https://ws4.sinaimg.cn/large/006tNc79gy1fh92aiohhuj319g0tk0wq.jpg)

我们通过此UI工具来新建数据库和查询measurements。

先为接下来的监控创建一个数据库

````sql
create database monitor
````

查询数据库下的measurements之前，我们需要先点击右上角的database下拉菜单来切换数据库。

查询measurements的脚本是

````sql
show measurements
````



### 安装采集器Telegraf

采集器telegraf也是influxdata出的开源产品，使用go语言编写，插件丰富，可扩展性强。

项目地址：https://github.com/influxdata/telegraf

我们下载Release出来的官方编译版本：https://github.com/influxdata/telegraf/releases

根据自己环境选择介质下载并安装。安装完毕之后，根据官方教程了解一下配置文件。我先给一个精简的配置文件，后续的插件功能可以在此基础上增加。

````shell
[global_tags]

# Configuration for telegraf agent
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = ""
  hostname = "192.168.37.1"
  omit_hostname = false

[[outputs.influxdb]]
  urls = ["http://192.168.37.10:8086"] # required
  database = "monitor" # required
  retention_policy = ""
  write_consistency = "any"
  timeout = "5s"

[[inputs.cpu]]
  percpu = false
  totalcpu = true
  fieldpass = ["usage_idle", "usage_iowait", "usage_system", "usage_user"]
  name_override = "cpu"

[[inputs.net]]
  interfaces = ["en0"]
  name_override = "net"
  fieldpass = ["bytes_recv","bytes_sent"]
````

各个字段的含义在telegraf工具生成的配置文件都有很详细的描述，挑几个比较重要的配置做二次理解。

- agent.hostname 采集器部署机器的地址，这是区分机器最重要的配置。
- inputs.*.name_override 数据表名，采集的数据会写入此表。

我们使用此配置启动采集器看看效果。

````shell
➜  temp telegraf --config ./telegraf-demo.conf start
2017-07-05T08:19:28Z I! Starting Telegraf (version 1.2.0-rc1-288-gf2bb4ac)
2017-07-05T08:19:28Z I! Loaded outputs: influxdb
2017-07-05T08:19:28Z I! Loaded inputs: inputs.cpu inputs.net
2017-07-05T08:19:28Z I! Tags enabled: host=192.168.37.1
2017-07-05T08:19:28Z I! Agent Config: Interval:10s, Quiet:false, Hostname:"192.168.37.1", Flush Interval:10s
````

启动日志能看出我们配置的outputs和inputs。

回到influxdb的AdminUI，查询measurements，可以看到我们已经采集到了CPU和网络的数据。数据间隔10秒，字段与采集器配置的fieldpass一致。

![](https://ws1.sinaimg.cn/large/006tNc79gy1fh939eu7a4j31kw138wpm.jpg)



### 安装Monitor UI工具Grafana

Grafana是一个开源的监控工具，不仅仅是UI那么简单。安装教程：https://grafana.com/grafana/download?platform=mac

选择自己的平台，跟着教程一点点安装即可。如果你用的是linux(centos)，使用 `systemctl start grafana-server` 启动grafana服务。

浏览器访问Grafana安装机器的3000端口打开Grafana，第一次访问时，会有个初始化工具的阶段，按照步骤一步一步来。

![](https://ws1.sinaimg.cn/large/006tNc79gy1fh93qrqeibj31i40dk760.jpg)

创建好dashboard，我们接下来配置监控图表。

#### 配置CPU图表

新建panel的主题是“Panel Title”，点击主题“Panel Title”，在弹出的Popup中选择“edit”，即可看到如下两张图的配置功能。

第一张图是图表的General选项。

第二张图是图表展示的核心配置，可以配置数据源，表，字段以及字段的计算函数等。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fh97fm3oxhj311y0gs41w.jpg)

- 修改图表主题为主机负载。
- 查询语句选择usage_user和usage_system两个字段，分别展示主机用户负载和系统负载。DataSource选择刚才初始化时添加的数据源。

配置完这两个tab之后，图表就会展示变化的曲线图了。

类似下图：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fh97o9792bj31b20hg41u.jpg)

在“Axes“标签页中可以修改左纵坐标的刻度单位，Unit选择"percent(0-100)"即可。

#### 配置网络图表

新建一个panel，用于配置网络IO的图表。

按照下图配置网络监控图表的数据源。

![](https://ws2.sinaimg.cn/large/006tNc79gy1fh981m6xyrj31aq0lqdj3.jpg)

我把出网和入网放在一张图上，当然也可以配置到两个独立的Panel，这样看起来就不会有干扰了。最终的网络速率曲线图如下：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fh9858nk9hj31as0hk40g.jpg)

到这里我们已经完成了CPU和网络的数据采集和曲线图配置展示。

接下来，我们准备用springboot快速开发一个简单的HTTP RESTful API，围绕它去做数据的采集和展示。

### 简易RESTful API工程

DEMO源码地址：https://github.com/0x0010/monitor-api-demo

系统只暴露了首页的API，并且使用Metrics实现了请求速率计算和访问次数统计。

````java
@RestController
class ApiIndexController {
  @RequestMapping(value = "/", method = RequestMethod.GET)
  public String index() {
    Monitor.idxTpsMark();
    try {
      Thread.sleep(50);
    } catch (InterruptedException ignore) {

    }
    return "Greetings from Spring Boot!";
  }
}
````

监控地址通过注册MonitorServlet暴露给采集器

````java
@Bean
public ServletRegistrationBean monitorServlet() {
  return new ServletRegistrationBean(new MonitorServlet(), "/monitor/*");
}
````

- 首页访问次数及TPS指标接口

  Path：/monitor/IdxTps

  接口返回值：{"idxCount":0,"idx15mRate":0.0,"idxMeanRate":0.0,"idx1mRate":0.0,"idx5mRate":0.0}

- JVM内存信息

  Path：/monitor/MemInfo

  接口返回值：{"heapMaxSize":3817865216,"heapSize":333971456}




### API Demo指标采集器配置

在telegraf.conf文件中继续追加input插件

````shell
[[inputs.httpjson]]
    name_override="index_tps"
    servers = [
       "http://127.0.0.1:8090/monitor/IdxTps"
    ]
    response_timeout="5s"
    method="GET"

[[inputs.httpjson]]
    name_override="jvm_memory"
    servers = [
       "http://127.0.0.1:8090/monitor/MemInfo"
    ]
    response_timeout="5s"
    method="GET"
````

我们使用influxdb的httpjson插件采集Demo暴露的两个指标接口。



### 配置应用指标展示

现在应用指标有三个：首页访问次数，首页地址TPS和Jvm内存信息。首页的访问次数我们可以配置成数字的形式展示，另外两个指标继续配置成曲线图表。

配置过程与cpu和网络的非常类似，最终的效果图如下

![](https://ws2.sinaimg.cn/large/006tNc79gy1fh9dgzg3i3j31kw08ujtu.jpg)

在配置首页请求次数时，Options的stat要选择Current，否则默认的AVG会造成图表数据不准。

![](https://ws2.sinaimg.cn/large/006tNc79gy1fh9diky9hsj31300h6acc.jpg)



至此，基于influxdb,telegraf和grafana的监控系统就开发完成了。

过程很简单，基本没有什么阻碍，当然还有更多的监控功能等待我们去发掘和学习。

