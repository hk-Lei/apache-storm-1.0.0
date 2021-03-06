---
title: Flux
layout: documentation
documentation: true
---

一个用于简化 Apache Storm 流式计算任务创建和部署的框架

## 定义
**flux** |fləks| _noun_

1. 流入流出的动作或过程
2. 持续变化
3. 在物理学中，流体、辐射、微粒可以穿过一定的区域
4. 与固体物质混合，用来降低熔点

## 基本原理
当配置是写死的时候总是会出现很多问题。不应该有人为了更改配置而重新编译或打包应用程序。

## 简介
Flux 是一个用于简化创建和部署 Apache Storm Topologies 的框架和工具集。


你是不是发现你以前经常重复写如下代码：

```java

public static void main(String[] args) throws Exception {
    // 决定我们是不是在本地运行的逻辑...
    // 创建必要的配置项...
    boolean runLocal = shouldRunLocal();
    if(runLocal){
        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology(name, conf, topology);
    } else {
        StormSubmitter.submitTopology(name, conf, topology);
    }
}
```

从来不会像这样简单：

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --local config.yaml
```

or:

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --remote config.yaml
```

另一个经常被提及的痛点：通常在 Java 代码中绑定 Topology 图，任何更改都需要重新编译和打包 Topology 的 jar 文件。Flux 旨在减轻这个痛点：通过将所有的 Storm 组件打包成一个 jar， 用一个额外的文本文件来定义 topologies 的布局和配置。

## 功能

 * 不用在 Topology 的代码嵌入配置项，易于配置和部署 Storm topologies (包括 Storm 核心和微批 API)
 * 支持已有的 topology 代码 （见下文）
 * 使用一个灵活的 YAML DSL 定义 Storm 核心 API (Spouts/Bolts)
 * YAML DSL 支持大多数 Storm 组件 (storm-kafka, storm-hdfs, storm-hbase, 等等)
 * 方便支持多语言的组件
 * 外部属性替换/过滤，对于配置/环境环境之间轻松切换(类似于 Maven 的 `${variable.name}` 替换方式)

## 使用

使用 Flux，需将其作为依赖打包到包含所有 Storm 组件的 fat jar 中，然后创建一个 YAML 文件来定义 topology 结构（参见下面的 YAML 配置项）。

### 从源码构建
使用 Flux 最简单的方式是，如下所述将其作为一个 Maven 依赖添加到项目中。

如果你想从源代码构建 Flux 及运行单元/集成测试，则需要在系统中安装以下软件：

* Python 2.6.x or 或更高版本
* Node.js 0.10.x or 或更高版本

#### 构建中运行单元测试

```
mvn clean install
```

#### 构建中不运行单元测试
如果想在构建 Flux 时不用安装 Python 或者 Node.js，你可以简单的跳过单元测试：

```
mvn clean install -DskipTests=true
```

注意：如果你计划使用 Flux 将 topologies 部署在远程集群上，那么仍然需要安装 Python，因为是 Apache Storm 必须的。

#### 构建中运行集成测试

```
mvn clean install -DskipIntegration=false
```


### Maven 打包
为了在 Storm 组件中启用 Flux，需要将其添加为一个依赖，使得其包含在 Storm topology jar 中。可以使用 Maven shade 插件（首选）或者 Maven assembly 插件（不推荐）来完成。

#### Flux Maven Dependency
目前版本(译注：1.0.0)的 Flux 已经在 Maven 仓库中可用，坐标如下：

```xml
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>flux-core</artifactId>
    <version>${storm.version}</version>
</dependency>
```

#### 创建一个启用 Flux 的 Topology JAR
下面的例子演示了通过 Maven shade 插件使用 Flux：

 ```xml
<!-- 在 shaded jar 中引入 Flux 和用户依赖 -->
<dependencies>
    <!-- 引入 Flux -->
    <dependency>
        <groupId>org.apache.storm</groupId>
        <artifactId>flux-core</artifactId>
        <version>${storm.version}</version>
    </dependency>

    <!-- 在这里添加用户依赖... -->

</dependencies>
<!-- 创建一个包含所有依赖的 jar -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>1.4</version>
            <configuration>
                <createDependencyReducedPom>true</createDependencyReducedPom>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.apache.storm.flux.Flux</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
 ```

### 部署运行 Flux Topology
一旦 topology 组件和 Flux 依赖打包到一起，就可以通过 `storm jar` 命令在本地或者远程运行不同的 topologies。例如：假设 jar 包叫 `myTopology-0.1.0-SNAPSHOT.jar`，可以使用下述命令在本地运行：

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml

```

### 命令行选项
```
usage: storm jar <my_topology_uber_jar.jar> org.apache.storm.flux.Flux
             [options] <topology-config.yaml>
 -d,--dry-run                 Do not run or deploy the topology. Just
                              build, validate, and print information about
                              the topology.
 -e,--env-filter              Perform environment variable substitution.
                              Replace keys identified with `${ENV-[NAME]}`
                              will be replaced with the corresponding
                              `NAME` environment value
 -f,--filter <file>           Perform property substitution. Use the
                              specified file as a source of properties,
                              and replace keys identified with {$[property
                              name]} with the value defined in the
                              properties file.
 -i,--inactive                Deploy the topology, but do not activate it.
 -l,--local                   Run the topology in local mode.
 -n,--no-splash               Suppress the printing of the splash screen.
 -q,--no-detail               Suppress the printing of topology details.
 -r,--remote                  Deploy the topology to a remote cluster.
 -R,--resource                Treat the supplied path as a classpath
                              resource instead of a file.
 -s,--sleep <ms>              When running locally, the amount of time to
                              sleep (in ms.) before killing the topology
                              and shutting down the local cluster.
 -z,--zookeeper <host:port>   When running in local mode, use the
                              ZooKeeper at the specified <host>:<port>
                              instead of the in-process ZooKeeper.
                              (requires Storm 0.9.3 or later)
```

**注意:** Flux 努力避免与 `storm` 命令行的冲突，允许使用 `storm` 命令的其他任何命令选项。

例如：你可以使用 `storm` 命令的 -c 选项去覆盖 topology 的配置项。下述示例命令将运行 Flux 并且覆盖 `nimbus.seeds` 配置：

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --remote my_config.yaml -c 'nimbus.seeds=["localhost"]'
```

### 简单的输出
```
███████╗██╗     ██╗   ██╗██╗  ██╗
██╔════╝██║     ██║   ██║╚██╗██╔╝
█████╗  ██║     ██║   ██║ ╚███╔╝
██╔══╝  ██║     ██║   ██║ ██╔██╗
██║     ███████╗╚██████╔╝██╔╝ ██╗
╚═╝     ╚══════╝ ╚═════╝ ╚═╝  ╚═╝
+-         Apache Storm        -+
+-  data FLow User eXperience  -+
Version: 0.3.0
Parsing file: /Users/hsimpson/Projects/donut_domination/storm/shell_test.yaml
---------- TOPOLOGY DETAILS ----------
Name: shell-topology
--------------- SPOUTS ---------------
sentence-spout[1](org.apache.storm.flux.spouts.GenericShellSpout)
---------------- BOLTS ---------------
splitsentence[1](org.apache.storm.flux.bolts.GenericShellBolt)
log[1](org.apache.storm.flux.wrappers.bolts.LogInfoBolt)
count[1](org.apache.storm.testing.TestWordCounter)
--------------- STREAMS ---------------
sentence-spout --SHUFFLE--> splitsentence
splitsentence --FIELDS--> count
count --SHUFFLE--> log
--------------------------------------
Submitting topology: 'shell-topology' to remote cluster...
```

## YAML 配置项

在 YAML 文件中定义（或描述） Flux topology，一个 Flux topology 由以下部件组成：

  1. 一个 topology 名称
  2. topology 的 "组件(components)" 列表(将在环境中可用的 Java 对象)
  3. **EITHER** (DSL topology 定义):
      * spouts 列表，每个需要一个唯一的 ID 标识
      * bolts 列表，每个需要一个唯一的 ID 标识
      * "stream" 列表，代表了在 spouts 和 bolts 间传输的 tuples 流
  4. **OR** (可用产生 `org.apache.storm.generated.StormTopology` 实例的 JVM 类):
      * 一个 `topologySource` 定义.

**译注：** 3、4 二选一。

例如，这有一个使用 YAML DSL 简单定义的单词统计的 topology：

```yaml
name: "yaml-topology"
config:
  topology.workers: 1

# 定义 spouts
spouts:
  - id: "spout-1"
    className: "org.apache.storm.testing.TestWordSpout"
    parallelism: 1

# 定义 bolts
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
  - id: "bolt-2"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

# 定义数据流
streams:
  - name: "spout-1 --> bolt-1" # 名称目前没有被使用 (logging、UI 等的占位符)
    from: "spout-1"
    to: "bolt-1"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: SHUFFLE


```
## 替换/过滤属性

通常开发者希望在不同配置之间切换自如，例如在开发环境和生产环境间的切换部署。这个可以使用不同的 YAML 配置文件来完成，但是这种方法导致了不必要的重复，尤其是在 Storm topology 没有改变的情况下，改变主机名、端口和并行度等配置。

在这种情况下，Flux 提供了属性过滤允许将不同的值外部化到不同的 `.properties` 文件中，在解析 `.yaml` 文件前替换他们。

使用 `--filter` 命令选项来启用属性过滤。例如，可以这样调用 Flux：

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml --filter dev.properties
```

使用如下的 `dev.properties` 文件：

```properties
kafka.zookeeper.hosts: localhost:2181
```

可以在 `.yaml` 文件中使用 `${}` 语法通过 key 引用那些属性值：

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "${kafka.zookeeper.hosts}"
```

这样，在解析 YAML 内容之前 Flux 将使用 `localhost:2181` 替换 `${kafka.zookeeper.hosts}`。

### 替换/过滤环境变量
Flux 也运行替换环境变量。例如，定义了一个环境变量名为 `ZK_HOSTS`，可以在 Flux YAML 中用如下的语法引用它：

```
${ENV-ZK_HOSTS}
```

## 组件

组件实质上是配置项中的 spouts 和 bolts 对象实例，如果熟悉 Spring 框架的话，组件可以类比 Spring beans。

每个组件至少有一个唯一的标识符（字符串）和类名(字符串)标识。例如，以下将创建一个 `org.apache.storm.kafka.StringScheme` 类的实例作为键 `"stringScheme"` 的引用，假设 `org.apache.storm.kafka.StringScheme` 类有一个默认的构造方法。

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"
```

### 构造参数、引用、属性和配置方法

#### 构造参数
可以通过在组件中添加一个 `contructorArgs` 元素来配置组件类构造方法的参数。 `contructorArgs` 是一个传递给类的构造函数的对象列表。下面示例中通过调用接收一个字符串的构造函数创建了一个对象。

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"
```

#### 引用
每个组件示例是由一个唯一的 id 标识的，其允许被其他组件使用/重用。引用现有的组件，需要使用 `ref` 标签指定组件的 id。

在下例中，创建了一个 id 为 `"stringScheme"` 的组件，然后作为另一个组件构造方法的参数被引用。

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme" # id 为 "stringScheme" 的组件必须在上面已经声明了
```
**注意:** 引用只能在被声明的对象下面使用。

#### 属性
除了支持调用不同参数的构造方法之外，Flux 还允许使用类似 JavaBean 的 setter 方法来配置组件，字段需声明为 `public`：

```yaml
  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"
```

在上述示例中，`properties` 声明会致使 Flux 在 `SpoutConfig` 中去寻找一个 public 方法 `setForceFromStart(boolean b)` 并尝试调用它。如果 setter 方法没找到，Flux 会再去寻找一个名为 `ignoreZkOffsets` 的 public 实例变量并尝试设置其值。

引用也可以作为属性的值。

#### 配置方法
从概念上讲，配置方法类似于属性和构造参数 -- 他们允许你在一个对象被构造后调用一个任意的方法。配置方法对于处理类是很有用的，不用暴露 JavaBean 方法和构造器就可以完全配置一个对象。常见的例子包括类都使用建造者模式 (`builder pattern`) 配置/构成。

下述的 YAML 示例创建了一个 bolt 并配置其调用一些方法：

```yaml
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.flux.test.TestBolt"
    parallelism: 1
    configMethods:
      - name: "withFoo"
        args:
          - "foo"
      - name: "withBar"
        args:
          - "bar"
      - name: "withFooBar"
        args:
          - "foo"
          - "bar"
```

相应的方法声明如下：

```java
    public void withFoo(String foo);
    public void withBar(String bar);
    public void withFooBar(String foo, String bar);
```

传递给配置方法的参数同构造函数参数工作方式相同，也支持引用。

### 在构造参数、引用、属性和配置方法中使用 Java 枚举 (`enum`)
在 Flux YAML 文件中可以很容易的使用 Java 枚举的值作为参数，只需引用 `enum` 即可。

例如，[Storm 的 HDFS 模块]() 包含以下枚举定义（简写）：

```java
public static enum Units {
    KB, MB, GB, TB
}
```

`org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy` 类有如下的构造方法:

```java
public FileSizeRotationPolicy(float count, Units units)
```

下面的 Flux `component` 定义可以被用来调用构造函数:

```yaml
  - id: "rotationPolicy"
    className: "org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy"
    constructorArgs:
      - 5.0
      - MB
```

上述定义的功能相当于如下的 Java 代码:

```java
// 当文件达到 5 MB 时切文件
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);
```

## 配置 Topology

`config` 部分是 Storm topology 配置参数中的一个简单的键值对，其将被作为 `org.apache.storm.Config` 类的示例传递给 `org.apache.storm.StormSubmitter`：

```yaml
config:
  topology.workers: 4
  topology.max.spout.pending: 1000
  topology.message.timeout.secs: 30
```

# 已存在的 Topologies

如果你有已存在的 Storm topologies，你也可以使用 Flux 来部署/运行/测试它们。这个特性允许你为现有的 topology 利用 Flux 构造函数参数、引用、属性和 Topology 配置声明。

使用现有的 topology 类最简单的方法是定义一个如下的 `getTopology()` 的实例方法:

```java
public StormTopology getTopology(Map<String, Object> config)
```
或:

```java
public StormTopology getTopology(Config config)
```

可以使用如下的 YAML 来配置 topology：

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
```

如果你想使用的 topology 类中有不同的方法名(即不是 `getTopology`)，你可以重写它:

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
  methodName: "getTopologyWithDifferentMethodName"
```

**注意：** 这个指定的方法必须接受一个 `java.util.Map<String, Object>` 或
`org.apache.storm.Config` 类型的参数，返回一个 `org.apache.storm.generated.StormTopology` 对象。

# YAML DSL

## Spouts 和 Bolts

YAML 配置中，Spout 和 Bolt 分别在他们自己的配置项中。`component` 的定义由 Spout 和 Bolt 的定义组成，通过添加 `parallelism` 参数来设置部署 topology 时组件的并行度。

由于 spout 和 bolt 的定义继承自 `component`，所以他们也支持构造函数参数、引用和属性。

Shell spout 示例：

```yaml
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.spouts.GenericShellSpout"
    # shell spout 的构造函数需要 2 个参数: String[], String[]
    constructorArgs:
      # 命令行
      - ["node", "randomsentence.js"]
      # 输出 fields
      - ["word"]
    parallelism: 1
```

Kafka spout 示例:

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme"

  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"

# 可选的 kafka 配置
#  - id: "kafkaConfig"
#    className: "org.apache.storm.kafka.KafkaConfig"
#    constructorArgs:
#      # brokerHosts
#      - ref: "zkHosts"
#      # topic
#      - "myKafkaTopic"
#      # clientId (可选)
#      - "myKafkaClientId"

  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"

config:
  topology.workers: 1

# 定义 spout
spouts:
  - id: "kafka-spout"
    className: "org.apache.storm.kafka.KafkaSpout"
    constructorArgs:
      - ref: "spoutConfig"

```

Bolt 示例:

```yaml
# 定义 bolt
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.bolts.GenericShellBolt"
    constructorArgs:
      # 命令行
      - ["python", "splitsentence.py"]
      # 输出 fields
      - ["word"]
    parallelism: 1
    # ...

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1
    # ...

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
    # ...
```
## 流和分组

Flux 中的流可以被描绘成 Topology 中 Spouts 和 Bolts 间的一系列连接 (图的边、数据流等)，其有一个相应的分组定义。

流定义包含下列属性：

**`name`:** 连接的名称(可选，目前未使用)

**`from`:** Spout 或 Bolt 源(发布者)的 `id`

**`to`:** Spout 或 Bolt 目的地(订阅者)的 `id`

**`grouping`:** 流分组定义

分组定义包含下列属性：

**`type`:** 分组类型： `ALL`、`CUSTOM`、`DIRECT`、`SHUFFLE`、`LOCAL_OR_SHUFFLE`、`FIELDS`、`GLOBAL` 或 `NONE`其中之一。

**`streamId`:** Storm 流 ID (可选。如果没指明将使用默认的 default )

**`args`:** 面向 `FIELDS` 分组的字段名列表。

**`customClass`** 面向 `CUSTOM` 分组的自定义分组类。

下例中的 `streams` 定义使用如下的连接建立了一个 Topology：

```
    kafka-spout --> splitsentence --> count --> log
```


```yaml
# 流定义
# 流定义就是定义 spouts 和 bolts 之间的连接
# 注意这种连接可以是周期性的
# 自定义流分组也是支持的

streams:
  - name: "kafka --> split" # 名称目前没有被使用 (logging、UI 等的占位符)
    from: "kafka-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```

### 自定义流分组
自定义流分组的定义通过设置分组类型为 `CUSTOM` 和定义一个 `customClass` 参数，告诉 Flux 如何自定义类实例化。`customClass` 定义继承自 `component`，所以它也支持构造函数参数、引用和属性。

下例中使用了一个自定义流分组类 `org.apache.storm.testing.NGrouping` 创建了一个流：

```yaml
  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: CUSTOM
      customClass:
        className: "org.apache.storm.testing.NGrouping"
        constructorArgs:
          - 1
```

## Includes 和 Overrides

Flux 允许包含其他的 YAML 文件，就像在同一个文件中定义一样。包含的可以是文件，也可以是 classpath 资源。

Includes 指定为一个键值对列表:

```yaml
includes:
  - resource: false
    file: "src/test/resources/configs/shell_test.yaml"
    override: false
```

如果 `resource` 属性设置为 `true`，通过 `file` 属性的值将加载 classpath 资源，否则其将被视为一个正常的文件。

`override` 属性控制如何有效的包含在当前文件中定义的值。如果 `override` 设置为 `true`, 解析时 `file` 文件中包含的值将会替换当前文件中的值； `override` 设置为 `false`, 当前文件中的值的解析优先级高，解析器将拒绝替换他们。

**注意:** Includes 目前还不是递归的，包含文件中的 Includes 将会被忽略。

## Word Count 示例

这个示例使用里一个 JavaScript 实现的 spout、一个 Python 实现的 bolt、和一个 Java 实现的 bolt。

Topology YAML 配置:

```yaml
---
name: "shell-topology"
config:
  topology.workers: 1

# 定义 spout
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.spouts.GenericShellSpout"
    # shell spout 的构造函数需要 2 个参数: String[], String[]
    constructorArgs:
      # 命令行
      - ["node", "randomsentence.js"]
      # 输出 fields
      - ["word"]
    parallelism: 1

# 定义 bolt
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.bolts.GenericShellBolt"
    constructorArgs:
      # 命令行
      - ["python", "splitsentence.py"]
      # 输出 fields
      - ["word"]
    parallelism: 1

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1

# 流定义
# 流定义就是定义 spouts 和 bolts 之间的连接
# 注意这种连接可以是周期性的
# 自定义流分组也是支持的

streams:
  - name: "spout --> split" # 名称目前没有被使用 (logging、UI 等的占位符)
    from: "sentence-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```


## 微批处理 (Trident) API

目前，Flux YAML DSL 只支持 Storm 核心 API，但对 Storm 微批处理的 API 支持在计划中。

如果在 Trident topology 中使用 Flux ，可以定义一个 topology getter 方法，在 YAML 配置中引用它：

```yaml
name: "my-trident-topology"

config:
  topology.workers: 1

topologySource:
  className: "org.apache.storm.flux.test.TridentTopologySource"
  # Flux 默认将会调用 "getTopology" 方法，下列方法名会覆盖它。
  methodName: "getTopologyWithDifferentMethodName"
```
