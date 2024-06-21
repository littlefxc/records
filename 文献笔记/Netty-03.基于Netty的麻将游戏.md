---
title: 基于Netty的麻将游戏
tags:
  - netty
status: Doing
createDate: 2023-04-14
---

# 基于Netty的麻将游戏

# 需求分析

## 需求分析

---

麻将，在中国的流行可以说是非常广泛的，同时，也是非常具有地方性特色的一种游戏，每个地方的玩法可能都会有一些差异，比如四川血流成河、贵阳捉鸡、广东推倒胡等。

当然了，我们不可能完整地做一个麻将游戏出来，我们的课程是 Netty，所以，主要还是说 Netty 如何在游戏中进行使用，当然了，也不会不做一个游戏出来，我们会做一个简化版的麻将，最起码保证可以正常进行游戏。

## 需求收集

---

需求收集就非常简单了，对于麻将类的游戏，最好的竞品就是腾讯的欢乐麻将了，另外，还有其他一些小厂开发的地方性特色的麻将，既然，我们要做简化版的麻将，直接拿腾讯欢乐麻将的规则来做减法就可以了。

对于腾讯欢乐麻将，我们可以直接在小程序中打开，点击 更多 -> 帮助 即可看到所有欢乐麻将的规则：

![Untitled](欢乐麻将.png)

我这里就不一一截图了，我简单列举一下麻将的普适性规则：

1. 麻将一般有四个玩家，个别地方可以支持三人或者二人；

2. 麻将一副牌有这么几种牌型：万条筒（序数牌）、东南西北中发白（字牌）、春夏秋冬梅兰竹菊（花牌），共 144 张牌；

3. 二张一样的牌叫对子，三张一样牌的叫刻子，四张一样牌的叫杠，碰出去的叫明刻，手里的叫暗刻，杠别人打出的牌叫明杠，先碰再杠叫补杠，自己摸了四张一样的叫暗杠；

4. 牌局开始会进行洗牌，即打乱牌序；

5. 然后定庄，定庄的规则每个地方不太一样，最常见的掷骰子，按点数算，有的地方掷两次，有的地方掷一次；

6. 接着摸牌，线下是从庄家开始摸，每次摸两墩牌，也就是四张，轮流摸三次后，庄家跳着摸两张，其它玩家各一张，也就是初始摸牌后，庄家有 14 张牌，闲家有 13 张牌，在游戏里，我们称作发牌，发牌的规则不用这么麻烦，直接给庄家发 14 张牌，闲家发 13 张牌即可；

7. 如果发完牌庄家直接胡了，叫天胡；

8. 如果庄家打完第一张牌，闲家胡了，叫地胡；

9. 对于每一次出牌，其他玩家可能有吃、碰、杠、胡、过几种操作，这些操作一般还会有个倒计时，到时间了玩家还没有操作，服务端自动帮玩家操作，多次超时，进入托管状态；

10. 当玩家手里的牌组成了（2+3xN）（3 表示顺子或刻子）的牌型时，则表示可以胡牌了，个别地方支持七对胡；

11. 当玩家手里的牌还差一张就可以胡了的时候，叫作听牌，同时，可以听 1 到多张牌，我见过最吊的，是可以听所有牌（带癞子的玩法），像腾讯欢乐麻将做得比较好，还支持提示听什么牌；

12. 如果胡其他玩家打出的牌，打出这张牌的人叫放炮；

13. 如果胡自己摸的牌，叫自摸；

14. 如果胡别人回头杠的牌，叫抢杠；

15. 如果胡别人开杠摸牌后打的牌，叫杠上炮；

16. 如果胡自己开杠摸的牌，叫杠上开花；

17. 胡牌之后服务器计算番型并发送结算消息给玩家，玩家可以选择离场或者继续；

18. 如果有玩家离场，则等待其他玩家加入，当满足房间人数时，又开始新一轮的游戏；

普适性的规则大概是这样，不过，像血流成河、血战到底等可以无限胡牌或者带癞子的玩法，它们的规则略有出入。

然后，再看看其它竞品，可以发现，除了上面的规则外，它们很多都支持创建房间这样的制度，这样就很方便亲朋好友一起玩了。

到这里，需求收集阶段基本上差不多了，下面就进入需求分析阶段了。

## 详细分析

---

上面介绍的普适性规则，对于我们学 Netty 这门课程来说，还是太多了，所以，结合课程情况，我对这些规则做了一些裁减，裁减后的规则如下：

1. 使用创建房间的方式游戏，在创建房间时可以选择多少人参与游戏，这样，后面调试的时候会比较简单；
2. 为了方便，只保留万条筒三种序数牌，每种序数牌有 1 到 9，各四张，也就是一共 108 张牌；
3. 定庄，直接让创建房间的人为庄家；
4. 开局发牌时庄家 14 张牌，闲家 13 张牌；
5. 可碰、可杠、可胡、可过，不支持吃；
6. 不支持倒计时；
7. 不支持提示听牌；
8. 没有番型，只按最简单的底分计算输赢分数，底分在创建房间的时候传入；
9. 关于胡牌算法，是一种机密，无法透露，这里使用随机数替代；
10. 一局游戏之后直接结束，不支持继续游戏；

裁减之后的规则就比较简单了，用一张图来表示大致流程：
![Untitled](麻将游戏流程图.png)

当然了，我们这里写的还算比较简单的，真正的需求还会写清楚什么情况下可以碰，什么情况下明杠，什么情况下暗杠，等等，不过，我相信大家都有一定了解， 这里为了节约篇幅，就不再赘述了。

## 可行性分析

---

通过了详细分析，我们拿到了需求，那是不是就代表这个需求就具有可行性呢？显然不是，想像一下，我们平时做的多少需求都是无意义的，所以，还要进行可行性分析。

对于可行性分析，站在个人的角度，本次实战项目能够提升个人对于 Netty 的理解，能够提升个人对于游戏开发的了解，是很有意义的，所以，是可行的。站在老板的角度，这里没有老板，所以，嗯，是可行的。

# 系统设计

根据软件开发的基本步骤，系统设计，我们将分成技术选型、领域模型设计、接口设计、部署架构设计等四个部分来完成。

## 技术选型

---

技术选型，毫无疑问，我们选择的是 Netty，但是，基于 Netty，我们还需要做一些事情来构建系统，这些事情包含**网络协议选型**、**数据协议设计**、**编解码设计**等。

### 网络协议选型

也就是说请求是通过什么方式在客户端和服务端进行通信，常用的网络协议选型有 HTTP、HTTP2、TCP、WebSocket、UDP 等，让我们来对比一下：

| 协议 | 层 | 长 / 短连接 | 基于 | 其它 |
| --- | --- | --- | --- | --- |
| HTTP | 应用层 | 短连接 | 数据流 | 使用广泛，也可支持长连接，但无法支持服务端主动通信客户端 |
| HTTP2 | 应用层 | 长连接 | 数据流 | 目前使用还不是很广泛 |
| WebSocket | 应用层 | 长连接 | 数据流 | HTTP 协议的升级版，可以实现长连接，使用比较广泛 |
| TCP | 传输层 | 长连接 | 数据流 | 可靠，但数据流无界，需要自己分割，Netty 默认的协议 |
| UDP | 传输层 | 无连接 | 报文 | 不可靠，需要自己实现可靠性保证，但基于报文，没有粘包半包的问题 |

通过对比，可以发现，HTTP 无法支持服务端主动通信客户端，对于我们是不合适的，HTTP2 虽然支持长连接，但使用还不是很广泛，WebSocket 和 TCP 比较符合我们的要求，UDP 不可靠，还得自己保证可靠性，所以，我们的首要选择在 WebSocket 和 TCP 两者之间，考虑到未来可能会做小程序或者 Web 应用，所以，选择 WebSocket 协议是最佳选择，但是，选择 WebSocket 增加了复杂度和开发成本，第一个版本，我想做的简单点，所以，我们直接使用默认的协议就好了，即 TCP。

> 这里，我想说一下 Socket，Socket 它本身不是一种协议，翻译为套接字，它是应用层和传输层之间的一个抽象层，把 TCP/IP 层的复杂操作抽象成几个简单的接口供应用层调用以实现进程在网络中的通信。从设计模式的角度来看的话，Socket 就是一种门面模式，它把复杂的 TCP/IP 协议族隐藏在 Socket 接口后面。
> 
> 
> 另外，WebSocket 与 Socket 没有任何关系，就像 JavaScript 与 Java 的关系一样，纯属借名造势而已。
> 

### 数据协议设计

主要是解决两个问题：**粘包半包问题和协议内容**。粘包半包的问题，在前面的文章 [[Netty-01.如何解决粘包半包问题]] 我们也讨论过，这里肯定**选择的是 长度 + 内容 的方式来解决**。协议内容的问题就不是那么好确定了，参考业界比较优秀的协议，比如 Dubbo 和 RocketMQ，一般来说，**数据协议都会分成消息头和消息体两个部分，消息体存储真正的数据内容，消息头存储着一些扩展信息，比如版本号、请求地址（操作码、命令字）、序列化方式、自定义的扩展字段等。**

根据我们的业务场景，即麻将游戏场景，在请求头里面，可以存储版本号、命令字、请求 ID 等三个信息，每个字段占用一个 int 类型，最后，我们的数据协议大概是长这样子：

![Untitled](数据协议设计.png)

这里，Length 我选择 2 个字节，因为根据评估，我的请求内容不会超过（2^16-1=65535）个字节。

好了，网络协议和数据协议我们都弄好了，下面就是如何编解码了。

### 编解码设计

根据前面的内容，我们知道，编解码分为一次编解码和二次编解码。一次编解码，是对数据协议的编解码，这个在数据协议设计里面已经处理了，就是解决粘包半包的方式，也就是 长度 + 内容 法**。二次编解码，是对 Java 对象的编解码，或者换个通俗的叫法为序列化 / 反序列化方式**，对于消息头，格式是固定的，我们直接从 buffer 中读取或写入就好了，对于消息体，常见的序列化方式有 XML、JSON、Java 序列化、Protobuf 等，前面的章节我们也对比过，对于游戏这种对性能要求极高的应用，选择 Protobuf 无疑是最佳选择，但是，第一个版本，我想简单点，所以，我们先选择 JSON 来作为序列化方式，同时，JSON 在易读性方面也很有优势，也便于我们调试代码。

上面，我们从网络协议、数据协议、编解码三个维度对 Netty 这种技术选型做了分析，对于任何网络应用都要经过这些步骤，可以说，是通用化的解决方案，但是，对于具体的业务场景来说，领域模型设计就比较个性化了，下面我们就来看看如何对麻将这种个性化场景做领域模型方面的设计。

## 领域模型设计

领域模型设计，在不同的场合，可能也会称作数据库设计，或者数据结构设计，不过，它们多多少少还是有一些区别的，同学们可以自行体会一下。

对于麻将游戏的场景，我先给出下面一段描述：

玩家 A 打开 APP，登录到游戏，选择创建四个人的房间，玩家 B、C、D 同样地打开 APP，登录到游戏，同时，选择加入到玩家 A 创建的房间，此时，房间满足四人条件，游戏自动开始，玩家 A 作为房间创建者，自动成为庄家，开始发牌，玩家 A 获得 14 张牌，玩家 B、C、D 获得 13 张牌，玩家 A 开始出牌，玩家 A 出完牌后，玩家 B、C、D 可能会进行碰、杠、胡等操作，如果没有玩家操作，轮到玩家 B 摸牌、出牌，如此往复，直到摸完所有牌或者有玩家胡牌为止，发送结算消息给所有玩家，牌局结束，玩家离场。

我们先抽出这段描述中所有的名词：玩家、APP、游戏、房间、庄家、牌、消息、牌局，然后去除一些干扰名词。

![Untitled](领域模型设计_名词.png)

还剩下玩家、房间、牌、消息这几个名词，它们只是我们寻找的领域对象，让我们来分析一下它们应该具有的属性：

### 玩家

1. 玩家登录游戏需要什么？用户名和密码。

2. 玩家在游戏的时候操作的是什么？牌列表。

3. 碰、杠的牌要不要记录？要记录，补杠的时候需要看碰的牌，以后结算的时候可能会用到杠的牌，而且通常会把碰杠的牌放在玩家面前。

4. 玩家在房间内的位置要不要记录？要记录，不记录的话，每次要获取某个玩家的位置只能遍历房间中的所有玩家。

5. 玩家打出的牌要不要记录？打过线上麻将的同学应该都知道，玩家打出的牌是摆在自己面前的，所以，打出的牌也是要记录一下的。

6. 结算的时候按什么维度？暂且按积分，类似于斗地主那样，赢了加几分，输了扣几分。

所以，玩家至少应该具有 用户名、密码、牌列表、碰的记录、杠的记录、位置、打出的牌、积分等几个属性。

![Untitled](领域模型_玩家.png)

### 牌

1. 牌有哪些分类？万条筒，不考虑东南西北中发白、春夏秋冬梅兰竹菊的情况下。
2. 每种分类有哪些值？万条筒有 1 到 9 九种数值。

所以，关于牌，其实就这两个属性：类型和数值。

![Untitled](领域模型_牌.png)

因为类型只有三种，数值只有九种，所以，我们使用一个 byte 就完全可以存储了，甚至还有点浪费，但是没办法，Java 中能操纵的最小单位就是 byte 了。

因此，关于牌，我们并不需要再单独定义一个类型了，直接用 byte 就可以了，不过，为了方便操作，最好定义一个工具类专门用来处理牌相关的信息。

### 消息

1. 消息是指什么？客户端与服务端之间的一次通信就是一次消息的传递过程，消息就是通信。

2. 通信又是什么？一次通信往往伴随着动作，比如，玩家登录，客户端发送登录请求给服务端，服务端处理完成响应客户端，这个过程是两次通信。

3. 动作具有什么特征？动作往往都是动词，所以，只要找上面描述中的动词就八九不离十了。

4. 动词有哪些？打开、登录、创建、加入、开始、成为庄家、发牌、获得、出牌、碰、杠、胡、操作、摸牌、发送、结束、离场。

5. 哪些动词不是客户端与服务端之间通信？打开。

6. 成为庄家是不是？成为庄家，服务端可以不给客户端发送请求，而且，我们这次的需求并没有使用到庄家，所以，暂且认为它不是。

7. 发牌是不是？发牌，发完牌之后，服务端要通知客户端哪个玩家拿了哪些牌或者多少张牌，所以，发牌可以认为是。

8. 获得是不是？同上，发完牌服务端通知客户端玩家获得了哪些牌，所以，获得也可以认为是。

9. 操作是不是？操作这个概念有点抽象，出牌、碰、杠、胡、摸牌都可以说是一种操作，所以，可以认为它是凌驾在这几个消息之上的消息，也就是抽象类的概念。

10. 发送是不是？发送本身不是消息，发送的东西才是消息，所以，发送不是。

11. 结束是不是？结束了，服务端要通知客户端，所以，结束，是。

12. 离场是不是？与结束一样，服务端要通知客户端玩家已离场。

经过上面的分析，还剩下 登录、创建、加入、开始、发牌、获得、出牌、碰、杠、胡、操作、摸牌、结束、离场。

这些消息还能不能再分类呢？

我认为可以分成三类：

1. 客户端请求服务端：登录、创建、加入。

2. 服务端通知客户端：开始、发牌、获得、结束、离场。

3. 包含以上两者，比如出牌先是服务端通知客户端，客户端玩家再出牌：出牌、碰、杠、胡、操作、摸牌。

那么，每一种分类内部还能不能再抽象呢？

我认为是可以的：

1. **开始、发牌、获得**，这三者都是在游戏开始的时候服务端主动通知客户端的，而且，是通知所有客户端，所以，这三条消息，我认为可以合并成一条，我们暂且称之为游戏开局通知。
2. **结束、离场**，在我们的需求中，一局游戏结束，整个房间的游戏就结束了，所以，暂且也可以把这两个消息合并成一个。
3. **出牌、碰、杠、胡、摸牌**，这些消息本身就是不同的操作，所以，我们可以把操作作为父类，把这几个消息作为子类来处理，其实还有个 “过” 的操作，即询问玩家碰不碰的时候，玩家不碰。
4. **另外，每一次操作之后都应该刷新牌局信息**，比如，哪张牌打出了，谁摸牌了，等等，这样做的好处是防止某个客户端网络不稳定，依赖于客户端收到出牌消息来刷新牌局可能出现错乱的情况，而这个牌局的刷新其实与游戏开局通知是一样的，都是把房间的完整信息传递给客户端，所以，这两个可以合并为房间刷新通知。
5. **最后，上面的描述最后还有一个结算消息**，这也是一个单独的消息。

综上所述，最后的消息有 登录、创建、加入、结束（离场）、操作（出牌、碰、杠、胡、摸牌、过）、房间刷新通知、结算等。

![Untitled](领域模型_消息.png)

到这里，领域模型设计到这里基本就完成，下面，我们再来看看接口设计。

## 接口设计

接口设计，一般是针对 Web 系统来说的，在 Spring MVC 中，一般是指 Controller 层的设计，这个阶段，可以在领域模型设计完毕之后，立马就开始，接口设计完成之后交给客户端，两边就可以一起开发了。

但是，我们这次的实战项目并不是 Spring MVC，接口设计在哪里呢？

其实，就是每一条消息的详细设计，在上面的领域模型设计中，我们已经归纳出了所有的消息类型，现在只需要把它们加上属性就可以了。真正意义上来说，上面的消息并不能算是领域模型设计，只是我们是第一次分析这个过程，所以，我觉得把它放在领域模型设计这一小节也是可以的，后面，我们熟悉了这个过程，可以直接把消息的设计拿到接口设计这里来。更通俗易懂地来说，在需求的描述中，名词一般就是领域模型，动词一般就是领域模型的行为，行为一般对应着接口。

好了，下面我们就来详细分析每一条消息应该具有的属性，为了方便，我们按照牌局进行的顺序来。

### 登录

登录分成登录请求和登录响应，登录请求需要输入用户名和密码，登录响应返回是否登录成功，同时返回玩家的信息，登录失败还需要有相应的提示，所以，对于登录，应当分化为两条消息：

![Untitled](领域模型_登录.png)

### 创建房间

根据上面房间的设计，应该在创建房间的时候传入人数和底分，还需要其它属性吗？

好像不需要了，需要响应吗？可以有响应，比如，服务端有检查人数和底分是否满足条件，此时，是需要给客户端一个响应告诉客户端是否创建房间成功的。

![Untitled](领域模型_创建房间.png)

### 加入房间

加入房间，比较简单了，输入房间号，如果房间没满就可以加入，如果房间满了就加入失败，所以，加入房间是需要一个响应告诉客户端是否加入成功的。

![Untitled](领域模型_加入房间.png)

### 房间刷新通知

房间刷新通知，顾名思义，就是将房间的信息发送给客户端，所以，直接把房间本身作为它的字段就可以了。

不过，有个问题，房间中是包含玩家信息的，每一个玩家是有牌列表的，所以，发送消息的时候要注意一下，针对不同的玩家看到的消息内容本身是不完全一样的，每个玩家都只能看到自己的牌列表，其他玩家的牌列表要隐藏起来。

那么，房间刷新通知在哪些情况下会触发呢？

1. 有个加入房间的时候；
2. 有人操作（出牌、碰、杠、胡、摸牌）的时候；

因此，还需要有个行为类型的字段，客户端拿到不同的行为类型做出不同的动画，比如，出牌就播放玩家出了一张牌的动画，摸牌就播放玩家摸了一张牌的动画。

另外，关于行为类型，除了加入房间，其它的都是牌局操作，所以，我们直接使用操作类型来表示行为类型，当没有操作类型的时候就表示为加入房间。

所以，房间刷新通知一共有两个属性：操作类型和房间。

![Untitled](领域模型_房间刷新通知.png)

### 操作消息

操作，在客户端与服务端之间是双向的，服务端首先通知客户端可以进行哪些操作（同时也要通知其他玩家），客户端操作完了，再通知回给服务端，最后，服务端再通知其他玩家，谁谁谁做了什么操作，所以，操作应当分成三种不同类型的消息：操作通知、操作请求、操作结果通知。

另外，针对不同的操作类型，需要传输的数据可能又不太一样，比如出牌通知和碰牌通知，出牌通知其他玩家是知道轮到谁出牌了的，碰牌通知其他玩家是不知道谁可以碰牌的，只有真正碰牌完毕了，其他玩家才知道，所以，这里根据不同的操作类型，又可以分化成不同的消息，比如出牌三种消息，碰牌三种消息，等等，不过，我们并不打算这么做，因为它们太像了，因为个别字段的不同就拆分成更细的消息，有点得不偿失，而且，非常不利于后期的扩展，比如，后面再加一种操作，吃，又要定义三种消息，这样就太费事了，所以，针对操作，我们一共只有三个消息，通过不同的操作类型来区分。

### 操作通知

经过上面的分析，操作类型是必不可少的，但是，有一种状况需要特别关注。

试想，玩家 A 出了一张牌，玩家 B 可能既能碰，也能杠，甚至还可以胡，这种情况下要怎么通知玩家 B 呢？

有两种方案，一种是通知里面维护一个操作列表，不过有点浪费空间，另一种方案是只使用一个 int，不同的位代表不同的操作，我们知道一个 int 有 32 位，所以，可以代表 32 种不同的操作，后期扩展也方便。

除了操作类型还需要什么字段呢？

对于出牌，还需要知道通知谁出牌，所以还需要一个位置的字段。

对于碰、杠、胡，还需要知道哪张牌触发的这些操作，所以还需要一个触发的牌的字段。

综上所述，操作通知一共需要三个字段：操作类型、位置、触发的牌。

![Untitled](领域模型_操作通知.png)

### 操作请求

操作类型也是必不可少的，还需要其它什么字段呢？

对于出牌，服务端需要知道哪个位置出了哪张牌。

对于碰、杠、胡，服务端也是需要知道操作了哪张牌，不过服务端似乎知道是哪张牌，因为是服务端通知客户端哪张牌触发的操作，所以，服务端应该找个地方记录这张触发的牌，放在哪里比较合适呢？我认为放在房间信息中再合适不过了。

所以，操作请求一共需要两个字段：操作类型和操作的牌。

![[操作请求.png]]
## 部署架构设计

有了上面的设计，研发同学就可以开工了，为什么在前期还要考虑部署架构的设计呢？

因为不同的部署架构，可能对代码的架构产生巨大的影响。

就拿我们这次的实战案例来说，如果只是单机部署，那部署架构就非常简单，可以说是没有，那么，如果改成多机部署呢？是不是需要一个网关层做流量转发？这个网关该如何设计？有没有现成的开源框架可以使用？
![部署架构设计.png](部署架构设计.png)
如果是传统的 Web 应用，我会跟你说，服务做成无状态的就可以了。

但是，我们这次做的是游戏，对于游戏应用，性能的要求非常高，所以，很多数据必须保存在本机的内存中，注意，是本机的内存中，而不是像 Redis 那种分布式缓存的内存中，因此，游戏应用的服务是不可能做成无状态的。

这时候网关就起到至关重要的作用了，对于不在牌局中的消息，玩家的请求随便分发到哪个服务都是可以的，但是，对于牌局中的消息，同一个房间所有玩家的所有消息必须分发到同一台机器的同一个 JVM，这样，这些消息才能共享内存来进行处理。但是，这样还不够快，有没有更快的方法呢？后面实现的时候我再告诉你。

当然了，对于我们本次的实战案例，暂且只考虑单机部署的情况。
# 协议实现，双端打通

上一章节，我们已经从技术选型、领域模型设计、接口设计、部署架构设计等各个方面将系统规划好了。

本章，我们就基于系统设计来实现我们的系统，对于本次实战项目的实现，我想通过下面四个部分来讲解：

1.  协议实现，双端打通：根据技术选型的设计，将协议和编解码实现，并打通客户端和服务端，这样在编写代码的过程中随时可以进行一些简单的调试。
2.  领域模型实现：主要是参考领域模型的设计，将这些领域模型都定义好一个个的 Java 对象，同时，我也会把消息的定义也放在这一节，当然了，我们这里使用的也是贫血模型，为什么使用贫血模型呢？彼时，我会详细说明。
3.  业务逻辑实现：主要是服务端如何对这些消息进行处理，如何设计线程模型，加速服务端处理等。
4.  Mock 客户端实现：编写一个 Mock 客户端，并自测，四个客户端一桌麻将，妥妥的。

## 协议实现

上一章节中，我们已经将协议规划好了，协议分为 Header 和 Body，Header 中主要存储版本号、请求 ID、命令字，这里的<font color=red>命令字又可以称为操作码或者序列化类型等，主要是用来反查 Body 的真正类型，对于每一条消息，它们的命令字必须保证唯一性。</font>

因此，我们可以定义协议如下：
```java
@Data
public final class MahjongProtocol {
    /**
     * 协议头
     */
    private MahjongProtocolHeader header;
    /**
     * 协议体
     */
    private MahjongProtocolBody body;
}

@Data
public final class MahjongProtocolHeader {
    /**
     * 版本号
     */
    private int version;
    /**
     * 命令字
     */
    private int cmd;
    /**
     * 请求ID
     */
    private int reqId;
}

interface MahjongProtocolBody {}

public interface MahjongMessage extends MahjongProtocolBody {}
```
### 编解码实现

![[编解码实现.png]]

我们一再强调，**编解码分成一次编解码和二次编解码，一次编解码是对粘包半包的处理，将字节流分割成一个一个的片段，二次编解码是将这些片段再转换成 Java 对象。**

![[编解码实现_1.png]]

对于本次实战项目，最终转换成的Java对象就是 `MahjongProtocol` 对象。

首先，我们要针对一次解码写相应的解码器，用的是 长度+内容 的实现方式，对于这种方式，Netty也提供了支持，即`LengthFieldPrepender` 和 `LengthFieldBasedFrameDecoder` 类，代码如下：

```java
public class MahjongFrameEncoder extends LengthFieldPrepender {
    public MahjongFrameEncoder() {
        // 长度字段占用两个字节
        super(2);
    }
}
public class MahjongFrameDecoder extends LengthFieldBasedFrameDecoder {
    public MahjongFrameDecoder() {
        // 2个字节表示最多可传输65535个字节的内容
        super(65535, 0, 2, 0, 2);
    }
}
```

代码非常简单。

对于解码器，在经历过一次解码之后，拿到的就是一个个完整的 MahjongProtocol 对象的字节流了，对于这个字节流，我们在编写相应的解码器将其转换成 MahjongProtocol 对象即可。

对于编码器，整个过程正好相反，先通过二次编码器，将MahjongProtocol 对象转换成字节流，再通过一次编码器将这个字节流添加上长度字段，然后在发送出去，在发送的时候，他可能会跟其它的字节流合并在一起发送出去，接收方拿到这串字节流，根据一次解码器在进行解码就可以了。

那么，二次编解码该如何编写呢？

因为，我们上面的定义的协议的 body 是需要先解出 header 中的 cmd，根据 cmd 找到body 的类型，才能解出 body，所以，在编码的时候也是一样，我们先编码 header，header 中就三个字段，手动编码即可，对于body，我们的第一个版本也非常简单，使用json序列化成字节流，所以，编码器如下所示：

```java
public class MahjongProtocolEncoder extends MessageToMessageEncoder<MahjongProtocol> {

    @Override
    protected void encode(ChannelHandlerContext ctx, MahjongProtocol mahjongProtocol, List<Object> out) throws Exception {
        // 调用分配器分配一个ByteBuf
        ByteBuf buffer = ctx.alloc().buffer();
        // 调用的协议的编码方法
        mahjongProtocol.encode(buffer);
        // 添加到out中
        out.add(buffer);
    }
}
public final class MahjongProtocol {
    // 协议的编码方法
    public void encode(ByteBuf buffer) {
        // 调用header的编码方法
        header.encode(buffer);
        // 将body序列化成字节流写入到buffer中
        buffer.writeBytes(JSON.toJSONString(body).getBytes(StandardCharsets.UTF_8));
    }
}
public final class MahjongProtocolHeader {
    // header的编码方法
    public void encode(ByteBuf buffer) {
        // 写入版本号
        buffer.writeInt(version);
        // 写入命令字
        buffer.writeInt(cmd);
        // 写入请求ID
        buffer.writeInt(reqId);
    }
}
```

这里，你可能不禁要问：为什么 header 单独写一个编码方法，而 body 不也写一个编码方法呢？

那是因为 header 类型只有一个，而 body 的类型是有很多个的，如果在 MahjongProtocolBody 接口中定义一个 encode () 方法，那么，所有的消息类型都要实现这个方法，而它们中的内容并没有什么差别。但是，header 就不一样了，如果后面 header 增加新的内容，把编码方法放在 MahjongProtocolHeader 类中，就只需要修改这一个类就够了，如果把 header 的编码方法去掉，放到 MahjongProtocol 类中，也是可以的，只是后面增加新的字段，要同时修改两个类才可以，就不够单纯了。

二次编码器的编写相对来说比较简单一些，可以看到，这里并没有太关心 body 的类型，解码器就不一样了，请看：

```java
public class MahjongProtocolDecoder extends MessageToMessageDecoder<ByteBuf> {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
        // 创建一个协议对象
        MahjongProtocol mahjongProtocol = new MahjongProtocol();
        // 同样委托给协议自己去解码
        mahjongProtocol.decode(msg);
        // 添加到out中
        out.add(mahjongProtocol);
    }
}
public final class MahjongProtocol {
    public void decode(ByteBuf msg) {
        MahjongProtocolHeader header = new MahjongProtocolHeader();
        // 解码header
        header.decode(msg);
        this.header = header;

        // 命令字
        int cmd = header.getCmd();
        // 根据命令字获取body的真实类型
        Class<? extends MahjongProtocolBody> bodyType = getBodyTypeByCmd(cmd);
        this.body = JSON.parseObject(msg.toString(StandardCharsets.UTF_8), bodyType);
    }

    private Class<? extends MahjongProtocolBody> getBodyTypeByCmd(int cmd) {
        // todo 这里该如何写？
        return null;
    }
}
public final class MahjongProtocolHeader {
    public void decode(ByteBuf msg) {
        // 读取一个int赋值给version
        version = msg.readInt();
        // 读取一个int赋值给cmd
        cmd = msg.readInt();
        // 读取一个int赋值给请求ID
        reqId = msg.readInt();
    }
}
```

二次解码的过程相对来说要复杂一些，先解出 header，从 header 中取出 cmd，根据 cmd 找到正确的 body 类型，再使用 JSON 反序列化为 body 对象，这里的难点在于如何根据 cmd 的值找到正确的 body 类型，我提供以下几种思路：

1.  使用 Spring 容器来管理这些消息类型；
2.  使用枚举类型来管理这些消息类型；
3.  使用一个全局 Map 来管理这些消息类型；

使用 Spring 容器的话相对来说要方便一些，不过要编写自定义的 BeanPostProcessor 或者 BeanFactoryPostProcessor 来处理 cmd 和消息类型之间的映射关系，容错率也相对高一些，不过要引入 Spring，不在本课程的讨论范围之内。

使用全局 Map 的话，何时初始化这个 Map，怎么初始化这个 Map，是个头疼的问题。

使用枚举类型的话，每次添加一个新的消息，都要记得在枚举类中添加一条记录，相对来说有点麻烦，不过好在也比较简单，也不用依赖其它组件，所以，我们这里暂且使用枚举这种方式来管理这些消息类型，请看：

```java
@Slf4j
@Getter
public enum MessageManager {
    HELLO_REQUEST(1, HelloRequest.class),
    ;

    private int cmd;
    private Class<? extends MahjongMessage> msgType;

    MessageManager(int cmd, Class<? extends MahjongMessage> msgType) {
        this.cmd = cmd;
        this.msgType = msgType;
    }

    public static Class<? extends MahjongMessage> getMsgTypeByCmd(int cmd) {
        for (MessageManager value : MessageManager.values()) {
            if (value.cmd == cmd) {
                return value.msgType;
            }
        }
        log.error("error cmd: {}", cmd);
        throw new RuntimeException("error cmd:" + cmd);
    }
}
```

在这里，我们定义了一个 HelloRequest 的消息，把它添加到枚举中，并给它分配一个 cmd，这样我们就可以根据 cmd 取出消息的类型了，这种方式的缺点有两个：一是新增的消息要记得在枚举中添加一次，二是 cmd 千万不能重复。

此时，我们再回过头去修改 MahjongProtocol 中的解码方法：

```java
public final class MahjongProtocol {
    public void decode(ByteBuf msg) {
        MahjongProtocolHeader header = new MahjongProtocolHeader();
        // 解码header
        header.decode(msg);
        this.header = header;

        // 命令字
        int cmd = header.getCmd();
        // 根据命令字获取body的真实类型
        Class<? extends MahjongProtocolBody> bodyType = getBodyTypeByCmd(cmd);
        this.body = JSON.parseObject(msg.toString(StandardCharsets.UTF_8), bodyType);
    }

    private Class<? extends MahjongProtocolBody> getBodyTypeByCmd(int cmd) {
        // 从MessageManager中获取
        return MessageManager.getMsgTypeByCmd(cmd);
    }
}
```

这样的话，以后如果更换 cmd 与消息类型的映射方式，只需要修改 `getBodyTypeByCmd()` 这个方法就可以了。

好了，到此，协议实现和编解码实现都搞定了，下面就是把服务端和客户端实现，来检验它们的实现是否可行了。

### 服务端实现

服务端实现就比较简单了，前面我们已经着重分析过了，直接把代码拿过来即可。

```java
public class MahjongServer {

    static final int PORT = Integer.parseInt(System.getProperty("port", "8080"));

    public static void main(String[] args) throws Exception {
        // 1. 声明线程池
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            // 2. 服务端引导器
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            // 3. 设置线程池
            serverBootstrap.group(bossGroup, workerGroup)
                    // 4. 设置ServerSocketChannel的类型
                    .channel(NioServerSocketChannel.class)
                    // 5. 设置参数
                    .option(ChannelOption.SO_BACKLOG, 100)
                    // 6. 设置ServerSocketChannel对应的Handler，只能设置一个
                    .handler(new LoggingHandler(LogLevel.INFO))
                    // 7. 设置SocketChannel对应的Handler
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            // 打印日志
                            p.addLast(new LoggingHandler(LogLevel.INFO));
                            // 一次编解码器
                            p.addLast(new MahjongFrameDecoder());
                            p.addLast(new MahjongFrameEncoder());
                            // 二次编解码器
                            p.addLast(new MahjongProtocolDecoder());
                            p.addLast(new MahjongProtocolEncoder());
                        }
                    });

            // 8. 绑定端口
            ChannelFuture f = serverBootstrap.bind(PORT).sync();
            // 9. 等待服务端监听端口关闭，这里会阻塞主线程
            f.channel().closeFuture().sync();
        } finally {
            // 10. 优雅地关闭两个线程池
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

### 客户端实现

客户端的实现是我们之前没有讲过的，我这里先给出代码：

```java
@Slf4j
public class MahjongClient {

    static final int PORT = Integer.parseInt(System.getProperty("port", "8080"));

    public static void main(String[] args) throws Exception {
        // 工作线程池
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(workerGroup);
            bootstrap.channel(NioSocketChannel.class);
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    // 打印日志
                    pipeline.addLast(new LoggingHandler(LogLevel.INFO));
                    // 一次编解码器
                    pipeline.addLast(new MahjongFrameDecoder());
                    pipeline.addLast(new MahjongFrameEncoder());
                    // 二次编解码器
                    pipeline.addLast(new MahjongProtocolDecoder());
                    pipeline.addLast(new MahjongProtocolEncoder());
                }
            });

            // 连接到服务端
            ChannelFuture future = bootstrap.connect(new InetSocketAddress(PORT)).sync();

            log.info("connect to server success");

            future.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

客户端因为不需要像服务端那样去监听网卡，所以，就不需要 ServerSocketChannel 以及 bossGroup 相关的配置了，相信有服务端编码过程的了解，对于这段代码，你一定可以驾轻就熟的，我们就不再赘述了。

### 双端打通

好了，服务端和客户端的代码都写好了，如何把它们连接起来呢？

其实，直接启动两者的 main () 方法，就可以成功连接起来了：

```java
22:38:14 [nioEventLoopGroup-2-1] AbstractInternalLogger: [id: 0x1cafd9a5] REGISTERED
22:38:14 [nioEventLoopGroup-2-1] AbstractInternalLogger: [id: 0x1cafd9a5] CONNECT: 0.0.0.0/0.0.0.0:8080
22:38:14 [main] MahjongClient: connect to server success
22:38:14 [nioEventLoopGroup-2-1] AbstractInternalLogger: [id: 0x1cafd9a5, L:/192.168.175.1:55463 - R:0.0.0.0/0.0.0.0:8080] ACTIVE
```

只不过目前两者还没有任何消息的通信，所以，还看不到任何的效果。

因此，我们还需要定义一对消息，让它们在客户端与服务端之间传递，同时，两端还需要定义各自的 Handler 来处理这一对消息。

这一对消息我们姑且称之为 HelloRequest 和 HelloResponse，HelloRequest 在上面我们已经提起过了，这里给出它的代码：

```java
@Data
public class HelloRequest implements MahjongMessage {
    private String name;
}
```

非常简单，HelloResponse 是对 HelloRequest 的回应，我们姑且给它一个 message 的字段：

```
@Data
public class HelloResponse implements MahjongMessage {
    private String message;
}
```

另外，记得在 MessageManager 中添加 HelloResponse 与 cmd 的映射关系：

```java
public enum MessageManager {
    HELLO_REQUEST(1, HelloRequest.class),
    HELLO_RESPONSE(2, HelloResponse.class),
    ;
}
```

好了，两个消息我们也定义好了，下面就是定义两个 Handler 分别用来处理 HelloRequest 和 HelloResponse 了，请看服务端的 Handler：

```java
@Slf4j
public class MahjongServerHandler extends SimpleChannelInboundHandler<MahjongProtocol> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MahjongProtocol mahjongProtocol) throws Exception {
        // 协议头
        MahjongProtocolHeader header = mahjongProtocol.getHeader();
        // 检查是不是HelloRequest的cmd
        if (header.getCmd() == MessageManager.HELLO_REQUEST.getCmd()) {
            // 强转
            HelloRequest helloRequest = (HelloRequest) mahjongProtocol.getBody();
            // 打印日志
            log.info("receive msg: {}", helloRequest);
            // 获取消息的内容
            String name = helloRequest.getName();
            // 构建响应
            HelloResponse helloResponse = new HelloResponse();
            helloResponse.setMessage("hello " + name);
            // 响应对应的协议头
            MahjongProtocolHeader outHeader = new MahjongProtocolHeader();
            outHeader.setVersion(header.getVersion());
            outHeader.setReqId(header.getReqId());
            outHeader.setCmd(MessageManager.HELLO_RESPONSE.getCmd());
            // 响应对应的协议
            MahjongProtocol out = new MahjongProtocol();
            out.setHeader(outHeader);
            out.setBody(helloResponse);
            // 写出
            ctx.writeAndFlush(out);
        }
    }
}
```

在服务端的 Handler 中，我们接收到请求之后，构造了一个响应并返回，代码看起来稍微啰嗦了一点，是因为，我们二次编解码针对的是 MahjongProtocol，所以，传递到 MahjongServerHandler 中的也是 MahjongProtocol，如果确定后续的业务逻辑处理不会再使用到 header，那么，也可以在这里统一处理 header，往后面传递的时候只传递消息，这一块我们在业务逻辑实现的小节再详细讲解。

再看客户端的 Handler：

```java
@Slf4j
public class MahjongClientHandler extends SimpleChannelInboundHandler<MahjongProtocol> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MahjongProtocol mahjongProtocol) throws Exception {
        // 协议头
        MahjongProtocolHeader header = mahjongProtocol.getHeader();
        // 检查是不是HelloResponse的cmd
        if (header.getCmd() == MessageManager.HELLO_RESPONSE.getCmd()) {
            // 强转
            HelloResponse helloResponse = (HelloResponse) mahjongProtocol.getBody();
            // 打印响应
            log.info("receive response: {}", helloResponse);
        }

    }
}
```

客户端接收到消息之后，判断是不是 HelloResponse 的 cmd，然后打印出响应内容。

然后，把这两个 Handler 分别加入到服务端和客户端的 pipeline 中，分别启动服务端和客户端。

效果似乎不是我们预期的那样，仔细检查，发现，没有发送 HelloRequest 的地方呀，那么，在哪里发送 HelloRequest 呢？

无疑，是在客户端，但是，是在客户端的哪里发送比较合适呢？

有两种方式，一种是放到 MahjongClientHandler 的 channelActive () 方法中，一种是在 MahjongClient 的 main () 方法中连接建立之后发送，两种方式的代码分别如下。

放到 MahjongClientHandler 中：

```java
public class MahjongClientHandler extends SimpleChannelInboundHandler<MahjongProtocol> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MahjongProtocol mahjongProtocol) throws Exception {
        // ...省略
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        HelloRequest helloRequest = new HelloRequest();
        helloRequest.setName("tt");

        MahjongProtocolHeader header = new MahjongProtocolHeader();
        header.setVersion(1);
        header.setReqId(1);
        header.setCmd(1);

        MahjongProtocol mahjongProtocol = new MahjongProtocol();
        mahjongProtocol.setHeader(header);
        mahjongProtocol.setBody(helloRequest);

        ctx.writeAndFlush(mahjongProtocol);
    }
}
```

放在 MahjongClient 中：

```java
public class MahjongClient {

    public static void main(String[] args) throws Exception {
        try{
            // ... 省略
            
            // 连接到服务端
            ChannelFuture future = bootstrap.connect(new InetSocketAddress(PORT)).sync();

            log.info("connect to server success");

            // 连接建立完成之后发送hello消息给服务端
            HelloRequest helloRequest = new HelloRequest();
            helloRequest.setName("tt");

            MahjongProtocolHeader header = new MahjongProtocolHeader();
            header.setVersion(1);
            header.setReqId(1);
            header.setCmd(1);

            MahjongProtocol mahjongProtocol = new MahjongProtocol();
            mahjongProtocol.setHeader(header);
            mahjongProtocol.setBody(helloRequest);

            future.channel().writeAndFlush(mahjongProtocol);

            future.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

两者二选其一即可。

此时，分别启动服务端和客户端，观察控制台日志，可以发现，很顺畅。

服务端日志：

```java
//...省略
23:34:00 [nioEventLoopGroup-3-5] MahjongServerHandler: receive msg: HelloRequest(name=tt)
//...省略
```

客户端日志：

```java
//...省略
23:34:00 [nioEventLoopGroup-2-1] MahjongClientHandler: receive response: HelloResponse(message=hello tt)
//...省略
```

至此，双端已经完全打通。

## 领域模型实现

## 业务逻辑实现

## Mock客户端实现