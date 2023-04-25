env
datasource
returns

/flink run -m hadoop102:8081 -c com.atguigu.wc.StreamWordCount ./FlinkTutorial-1.0-SNAPSHOT.jar

会话模式下，集群的生命周期独立于集群上运行的任何作业的生命周期，并
且提交的所有作业共享资源。而单作业模式为每个提交的作业创建一个集群，带来了更好的资
源隔离，这时集群的生命周期与作业的生命周期绑定。最后，应用模式为每个应用程序创建一
个会话集群，在 JobManager 上直接调用应用程序的 main()方法

YARN 上部署的过程是：客户端把 Flink 应用提交给 Yarn 的 ResourceManager,
Yarn 的 ResourceManager 会向 Yarn 的 NodeManager 申请容器。在这些容器上，Flink 会部署
JobManager 和 TaskManager 的实例，从而启动集群。Flink 会根据运行在 JobManger 上的作业
所需要的 Slot 数量动态分配 TaskManager 资源

Standalone 模式中, 同时启动多个 JobManager, 一个为“领导者”（leader），其他为“后备”
（standby）, 当 leader 挂了, 其他的才会有一个成为 leader。
而 YARN 的高可用是只启动一个 Jobmanager, 当这个 Jobmanager 挂了之后, YARN 会再次
启动一个, 所以其实是利用的 YARN 的重试次数来实现的高可用。


使用yarn、应用模式

代码->数据流图->作业图->执行图


```text
客户端
JobManager
    分发器
    JobMaster
    ResourceManager
TaskManager
```

Flink 程序都可以归纳为由三部分构成：Source、Transformation 和 Sink

并行度设置

由于 keyBy 不是算子，所以无法对 keyBy 设置并行度


