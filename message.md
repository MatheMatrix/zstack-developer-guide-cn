# 消息
## 事件驱动架构

在微服务中，服务之间的通信通过消息完成，不存在直接的函数调用。在ZStack中也**大体如此**，服务间通过一个称为`CloudBus`的消息中间件相互通信。

>在后面的插件框架章节你会看到ZStack的服务也可以通过被称为`extension point（扩展点）`的机制单向调用

模块间函数式调用是程序紧耦合的一个元凶，这在传统的分层结构程序中经常出现。当程序规模到达一定程度，程序层级结构越来越深，代码耦合越来越紧，成为剪不断理还乱的Spaghetti Code，就如下图一样：
![](spagheticode.jpg)

解决的方法是将程序扁平化，变树形拓扑为星形拓扑。模块间并不直接相互调用，统一通过中央的消息总线交换消息，业务流程通过消息驱动，模块间甚至没有编译依赖，这就是我们所熟知的事件驱动架构（Event-driven Architecture)。

![](messagebus.png)

事件驱动架构天然具备分布式特性，只要消息总线可以跨进程跨机器通信，调用者和被调用者可以位于不同进程不同机器。这使得ZStack能够非常容易实现横向扩展，服务相互调用时，调用者并不关心被调用者是否在本地进程还是远端进程，一切由管理节点进程自带的*一致性哈希环*决定。

![](multinodes.png)
> **在分层架构中使用事件驱动**
> 
> 分层架构是最为常见的软件设计架构，在通常的面向对象设计中，软件通常分为：展现层（Presentation Layer）、应用层（Application Layer）、Business Layer（业务层）、Data Access Layer（数据层）。分层架构的的弱点在于层次之间采用函数直接调用，容易导致紧耦合产生巨石程序（Monolithic Application）。解决的方法是在层级之间使用事件驱动架构交互。
>
>微服务架构兴起后，由于其大量使用事件驱动架构，导致很多程序员认为事件驱动架构只存在分布式程序中，而且必须依赖消息中间件。事实并非如此。事件驱动架构也可以存在于单进程应用，并且不依赖于任何消息中间件。Linux的X11 server就是一个典型的例子。单进程事件驱动架构通常包含一个event loop作为事件分发引擎，程序的不同模块在event loop上注册处理事件的回调函数，在调用其他模块时通过向event loop发送消息，而不是直接调用其他模块的函数。这里event loop就相当于消息中间件，通过IO复用（IO multiplexing，在Linux中为select/poll系统调用）技术实现。
>
>自从9年前在MeeGo（Nokia和Intel的一个失败的移动互联网操作系统）的sensorframework项目中开始使用事件驱动架构起，我已在多个项目中成功使用了该架构，无论他们是单进程的还是分布式的。目前看来，事件驱动架构是非常有效的解耦程序的方法。

## 消息总线

消息总线是ZStack的神经中枢，所有服务都依赖消息总线进行相互调用。在选择消息中间件时，我考察了JMS协议、AMQP协议和Zeromq，最终选择了基于AMQP协议的Rabbitmq。当时的考虑是zeromq还不成熟，提供的机制比较底层，不易于使用；JMS虽然成熟，但跟Java绑定，不符合ZStack消息总线于语言无关的原则，所以最终选择了AMQP协议的Rabbitmq。

Rabbitmq可以负担20,000/s ~ 100,000/s的消息负载，并且提供灵路由的路功能，生态中的插件和外围功能也非常齐全，完全满足ZStack控制面发送消息的要求。与OpenStack不同，ZStack并没有将所有的agent通信都用Rabbitmq，也没有使用dynamic queue这类资源消耗的技术，而是采用了static queue的方式，并且每个服务只会创建一个queue。在一个双管理节点的部署中，queue的总数不超过100个。

ZStack的这种设计主要是考虑了在超大规模环境中，消息总线的性能和稳定性问题。设想一个拥有10万物理机的数据中心，如果每个物理机上的agent都接入同一个rabbitmq，将会产生10万个queue。在实际测试中，rabbitmq在创建到4万个queue时变得基本不可用，并且有内存泄露发生，重启机器后rabbitmq无法启动。为此，ZStack只是将rabbitmq用于服务间通信，服务与物理机上agent采用HTTP通信。HTTP无状态、connection on demand的特性非常适合大规模分布式场景，且各个语言都有非常成熟的HTTP库供使用。

ZStack消息总线包含两个Rabbitmq exchange，`BROADCAST`和`P2P`，前者用于广播，后者用于服务间点对点通信。每个服务都有唯一的queue与之对应，queue的命名规则为：
```
zstack.message.服务名字.管理节点UUID
```
例如`zstack.message.cluster.3a8e2ef4ab264e9caad8870b233437b2`。

在著名的《Enterprise Integration Pattern》中描述了一种dynamic channel（这里的channel对应rabbitmq的queue），其工作原理是消息的发送者在发送一条消息(request)前创建一个临时queue，并把该queue作为消息回复（reply）的返回地址设置在消息中，接收者处理完消息后将回复发送到临时queue中，发送者处理完回复后动态销毁该临时queue。也就是说，每一次消息通信都会有一个临时queue创建/销毁的动作。程序员的直觉告诉我这种教科书式的设计在真实系统中会带来非常大的系统负载，从而导致不稳定。很不幸，OpenStack的oslo.messaging中大量使用了dynamicc queue，导致其消息系统在高负载时非常不稳定。ZStack中的所有queue都是静态的，在高负载情况下（并发数万API）也能稳定工作。

>开发者可以在安装rabbitmq的机器上运行`rabbitmqctl list_queues`和`rabbitmqctl list_exchanges`查看ZStack所创建的queue和exchange

## 消息类型

下图是ZStack中消息继承关系总览：

![](message.png)

ZStack中所有消息（包括request和reply）都源于一个根class: [Message.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/Message.java)，它派生出的子类又包含下列几大类：

* **[Event.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/Event.java)**: 事件，用于广播，所有订阅该事件的服务都能收到该事件的一份拷贝
  * **[APIEvent.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/APIEvent.java)**： 代表API返回的event
  * **[LocalEvent.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/LocalEvent.java)**：除API返回外的其它事件
* **[NeedReplyMessage.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/NeedReplyMessage.java)**：需要回复消息请求，用于服务之间点对点通信
  * **[APIMessage.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/APIMessage.java)**：代表API的消息请求
  * **Others**：需要回复的非API消息
* **[MessageReply.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/MessageReply.java)**：消息回复
  * **[APIReply.java](https://github.com/zstackorg/zstack/blob/787402c53d9749ab6e18add656d797750549ea82/header/src/main/java/org/zstack/header/message/APIReply.java)**：API回复
  * **Others**：非API回复
* **Others**：所有不需要回复的非API消息


>**类型信息 —— 编译器的礼物**
>
>当你浏览ZStack消息的继承关系时，将发现很多消息继承了它的父类但却没有增加任何字段。例如APIReply.java：
>
>     package org.zstack.header.message;
>
>     public class APIReply extends MessageReply {
>     }
>  
>继承了父类MessageReply.java，但并没有定义额外字段，看似跟父类完全一样，有些多此一举。实际上APIReply.java包含了一个非常重要的隐藏内容：类型信息。由于该信息的存在，所有继承APIRely的class都自动具有了作为API回复的身份，无需我们添加额外字段说明。
>
>类型信息是编译器和面向对象编程送给程序员的礼物，它让我们无需在class中定义一个type字段来说明该class的用途。除了使用继承，在Java中还可以通过implement空interface给class添加多个类型信息。善用这些手段可以给OOP编程带来极大的方便，我们在后续的章节讲到服务入口时就会看到。

### 按发送方式分类

根据发送方式的不同，消息可以分为event和非event两种。对于event类消息，其发送接收方式为publish/subscribe，对该event感兴趣的服务在消息总线上订阅该event，发送者只需将event提交到总线，所有订阅者都会收到一份event的拷贝。例如

```java
APIDeleteEipEvent evt = new APIDeleteEipEvent(msg.getId());
bus.publish(evt);
```
在ZStack中event类型主要有两种：用于API返回的`APIEvent`和通报内部事件的·CanonicalEvent`。

非event类消息使用点对点通信，发送者需要在消息中指定接收者的`serviceId`，接收者在处理完消息后通过一个`MessageReply`返回结果给发送者。非event类消息是ZStack中使用最多的消息，主要用于服务间通信以及阻塞式API。

### 按是否需要回复分类

绝大多数ZStack消息请求需要回复，这类消息必须继承父类`NeedReplyMessage`。服务在处理完类型为`NeedReplyMesssage`类消息后必须发送一个回复，通常`MessageReply`的子类或者`APIEvent`的子类。

不需要回复的消息请求可以直接继承父类`Message`。这类消息主要用于要求接收者执行某项操作，但发送者并不关心操作执行的结果。例如`ReturnPrimaryStorageCapacityMsg`用于向primary storage归还容量，但发送者并不关心执行结果，因为primary storage服务应该保证容量归还始终成功。

### 按是否为API分类

继承父类`APIMessage`的消息带有API属性，它们跟非API消息最大的不同是，API消息的回复可以是一个event类消息（APIEvent）或一个非event类消息（APIReply），取决于API是阻塞类API还是非阻塞类API。

阻塞类API通常是*读*API，例如`APIQueryVmInstanceMsg`，调用者通常需要得到API的返回结果，才能执行后续的逻辑。阻塞类API的回复是一个`APIReply`，由API的执行者直接发送给API的调用者。

非阻塞类API通常执行某个操作，例如`APIStartVmInstanceMsg`，调用者无需等待API返回也可以执行后续逻辑。API执行者在完成操作后，通过一个`APIEvent`异步通知调用者。由于返回结果是一个event类消息，除API调用者外，其他服务也可以订阅API event来获知API执行的结果，例如对API执行结果进行审计的服务。

## 消息结构

通过继承关系，每种类型的消息都可以添加自己特定的字段，例如下面的这个例子：

![](createzonemessage.png)

`APICreateZoneMsg`消息具有多层继承关系，故除了包含自身定义的`name`和`description`两个字段外，还包含APIMessage特定的字段`session`以及根类`Message`的若干字段。我们来看看根类字段的定义：

| 字段| 描述 |
| --- | --- |
| **id**| UUIDv4字符串，唯一标识一个message |
| **serviceId** | 接收服务的名称 |
| headers | metadata，消息总线使用的内部信息，用户无需关心 |
| timeout | 超时时间，单位毫秒 |
| createdTime | 创建时间，Unix Epoch Time，精确到毫秒 |

其中`id`和`serviceId`两个字段最为重要，它们决定了消息如何发送和接收，在后面的小节中会具体介绍。

如上图所示，ZStack的消息以JSON文本的形势在总线上传递，其结构是一个嵌套的map：`Map<String, Map<String, Object>>`，最外层的的map只有一个元素，key是消息的Java全名，value是一个代表消息body的map。这样设计是为了让消息的接收者可以通过消息的Java全名将JSON文本恢复成对应的Java Object。

>**JSON: 丢失的信息**
>
>要问我写程序最讨厌的事情，那一定是参数传递。我一直认为参数传递的程序BUG一大源头。参数传递的路径越长、层级越深，程序的BUG就越多。我们的程序有一大半时间是在各种层级之间传递参数，或者是跟外部进程交换参数。每一次传递通常都带有一定程度的信息损失，因为你不大可能在各个层级间传递同样的参数，都会有某种程度的提取、封装、打包等动作，传递给一个层级的参数往往是这个层级所需要知道的最小信息，而不是全部信息。这里的信息损失一方面是人为造成的，例如一个读数据库的API，其身份验证完成后，传递给数据库层的参数通常就不再需要包含身份信息了；另一方面是数据通信格式造成的，JSON就是个典型。
>
>JSON是一种弱类型的数据格式，本身并不带类型信息。对Java这样的强类型语言就比较痛苦了，因为一个Java对象转换成JSON文本后会丢失所有的编译器信息，包括最重要的类型信息。这就导致Java库（例如GSON）在将一个JSON文本还原成对象时，在不指定对象类型的情况下只能还原成一个Map。在指定对象类型的情况下，JSON文本可以还原成该类型对应的对象（这里就要表扬一下Java的GSON库了，至少没有强迫程序员为每个用户类定义一个decoder。相反Python作为弱类型语言，其默认的Json库居然要求给每个用户类写decoder，否则只能还原成dict类型）。所以在ZStack的消息结构中，我将消息的class name编码到了JSON文本中，这样我们就可以把JSON文本还原成对应的消息对象了。例如在上面的图中，该JSON文本会被还原成*org.zstack.header.zone.APICreateZoneMsg*对象。
>
>但故事到此并没有结束，编码class name到JSON文本只解决最外层对象的类型问题，当对象内部包含List, Map这样的集合，或者包含带继承信息的成员变量时，这些类型信息都将丢失。例如下面这个例子：
>
>        class Parent {
>            int a;
>        }
>
>        class Child extends Parent {
>            String name;
>        }
>
>        class JSONObject {
>            List parents;
>            Parent child;
>        }
>
>        JSONObject json = new JSONObject();
>        json.parents.add(new Parent());
>        json.child = new Child();
>        
>        String jsonString = toJsonString(json);
>        JSONObject newJsonObject = toJsonObject(jsonString, JSONObject.class);
>在这个例子中，我们将一个`JSONObject`对象转换成JSON文本再还原回来，其成员变量的类型信息就全部丢失了，`parents` List中包含的不再是一个`Parent`对象，而是一个Map；`child`字段包含的也不是`Child`对象，而是它的父类对象`Parent`。Java对于这个问题并没有什么好办法，我每隔一段时间就会以`java json type info`，`java json schema`等关键字google，看有无新技术出现。到我写这篇文档为止，no luck。（如果你有什么好办法，一定要告诉我）。
>
>Java处理这种问题有两个办法，一是为这样的class写decoder，太繁琐，违背懒是科技进步第一动力的原则；第二是使用Jackson这样的库，在父类上使用annotation指明它可能有哪些子类，但这又违反了信息至上而下的设计原则，即父类的作者是不应该也不能够预测它会有哪些子类的。所以我自创了第三种方法，在Object转换成JSON文本时将成员变量的类型编码进去，例如：
>![](json.png)
>这里我们在消息的`header.schema`部分存放成员变量的类型信息，可以看到`inventory`是*org.zstack.header.vm.VmInstanceInventory*，`inventory.vmNics`是一个list，其第一个元素是*org.zstack.header.vm.VmNicInventory*类型，`inventory.allVolumes`的也是一个list，第一个元素是*org.zstack.header.volume.VolumeInventory*类型。这样在还原JSON文本时，我们就可以知道List，Map内部对象的类型，也可以保留继承对象的类型信息。
>
>故事还是没有结束，这种方法无法处理Set这样的无顺序集合，因为生成schema的时无法用下标[0],[1]...[n]编码元素位置。所以在ZStack中, 消息中不能使用Set类型。

## 一致性哈希环

状态交换是分布式程序头痛的问题之一，它往往制约了分布式集群的规模。ZStack的多节点部署也存在这样的问题，服务主要面临两种状态：

1. 消息应该发送给哪个服务？
2. 服务自己应该管理多少资源？

第一个问题源于多管理节点时，同样的服务在不同的机器让拥有多个实例，例如两个管理节点就会有两份相同的虚拟机服务。当一个服务向另一个服务发送消息时，首先要解决的问题是：**消息应该发送给哪个管理节点上运行的服务实例？**因为有多个服务实例时，每个实例应该只管理一部分资源。例如系统中总共1000个虚拟机，两个虚拟机服务就应该各管500个虚机，如果共管1000个就可能引起冲突，例如一个服务在启动一个虚拟机的时候另一个服务执行了停止该虚拟机的操作。虽然引入锁机制可以解决这个问题，但在后面的章节你会看到，ZStack的全异步架构是不允许业务逻辑中有锁的。这就引出了第二个问题：**服务实例之间应该如何协调分配管理资源？**，例如当一个新的管理节点启动后，其包含的虚拟机服务是否需要跟已有的服务协商，划分一部分虚拟机给新服务管理。

这些就是状态。在一个集群中不断交换这些状态会导致不稳定，不断的查询这些状态则会增加代码的复杂度，甚至引发性能问题。ZStack用一致性哈希环(Consistent Hash Ring)解决这个问题。

> **[一致哈希](https://zh.wikipedia.org/wiki/%E4%B8%80%E8%87%B4%E5%93%88%E5%B8%8C)**
> 
> 一致哈希 是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对**K/n**个关键字重新映射，其中**K**是关键字的数量， **n**是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

具体的介绍请参考上面的wiki链接。在ZStack，我们通过一致性哈希算法动态的算出一个资源是被哪个服务实例管理的，这样就同时解决了前面的两个问题：消息的发送者无需知道消息应该发送给哪个服务实例，因为哈希算法会算出；服务实例调用哈希算法就能知道哪些资源属于自己管理，当有服务实例加入或退出时，已有服务无需做任何状态交换，哈希环会动态扩张或收缩。也就是说，任何时候服务调用哈希算法都可以知道在当前时间点哪些资源由自己管理。

ZStack使用资源的UUID作为hash key，管理节点UUID作为hash value，只要知道资源的UUID就可以算得一个管理节点的UUID，这样就知道了该资源应该被哪个管理节点上的服务实例管理，也就知道了消息应被发送到哪儿。

![](hashring.png)
如上图所示，对同一个VM进行并发操作，消息的目的地都会是同一个管理节点上的虚拟机服务，这样该服务就可以使用后面章节介绍的队列来同步操作，从而避免了锁的使用。


