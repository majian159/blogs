# 能收获什么？
1. 更加了解TCP协议
2. Redis与客户端关闭连接的机制
3. 基于Apache Common连接池的参数调优
4. Linux网络抓包
# 情况简介
近期迁移了部分应用到K8s中，业务开发人员反馈说，会发现频繁出现 :  `redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.`  
堆栈如下图：
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6pus89lcjj219s08g0w8.jpg)
发生这个问题的应用的环境如下：
- Java8
- Jedis 2.9.0
# 排查
由于开发人员说近期才出现这个情况，我们首先怀疑是不是K8s环境的问题，进行了一轮K8s的网络环境问题排查。  
我们首先利用tcpdump在node节点和容器内进行抓包。

```sh
tcpdump -i <interfaceName> -C 100 -s0 -n -w node.pcap tcp
```

不出意外我们确实发现了大量由Redis服务器响应给客户端的RST（TCP Reset）包，连接重置。  
至此我们还是怀疑是网络不稳定引起的。  
我们搜索了TCP RST相关内容，可以看到RST一般由下列的几个情况引起：
- 到不存在的端口的连接请求
- 异常终止一个连接
- 检测半打开连接

## 极客的Redis，不按规矩出牌的"RST"？
随后我们又对网络又进行了几轮的测试。  
突然觉得有点不对劲，我们点开了RST包之前的包查看了包的内容。  
结果发现大部分是客户端发起`quit`数据，Redis服务端返回`ok RST`。  
虽然包的状态是`RST`，但包内容确又是跟商量好的一样是正常的"客户端说退出，服务端说ok"。   
带着这个问题，我们查了下Redis的相关文档，确实在默认情况下Redis是这么约束的，quit之后返回一个`RST`包。  
按照常理当连接不需要在使用的时候应该关闭连接，这种情况不是应该是我们理解的"TCP的4次挥手"来进行这个连接的告别（关闭）仪式吗？
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6wrum8do3j20jj0bfjsn.jpg)
## 为什么Redis的连接关闭使用"RST"？
我的猜想是 **不进行繁杂的4次挥手来提升性能。**  
这么做的好处是避免了4次挥手。
在网络情况差，客户端不稳定等情况下，能极大减少time_wait、close_wait等问题。  
Redis利用了TCP机制重新约束了客户端和服务端来进行连接关闭，流程如下。
- 客户端发送 "quit"
- 服务端返回 "ok" + RST
- 服务端直接强制关闭连接
- 客户端收到 "ok" 后自己关闭这个连接，并自己保证后续不在使用这个连接进行通信

## 既然返回"RST"是正常的，那么是哪里出了问题呢？
我们根据堆栈抛出的时间具体查看对应的RST包后发现，这种RST的情况与上面的不一致，这一次客户端发送的并不是 "quit" 数据，而Redis确返回了 `RST`。  
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6puxj1msvj21wy0ysgwf.jpg)
这时我觉得是Jedis这边的问题，去看了Jedis的release notes和issue，发现并没有相关的BUG。  
我们重新看一下TCP流的记录，发现这一次交互间隔了9分钟，最后一次交互为04:10:59秒的keepalive包，而发生问题的包的时间为04:19:57，我们又把返回`RST`可能的原因放在了"检测半打开连接"这点上。  
我又一次查看了业务使用的场景，发现了JedisPool按如下情况设置：
```java
config.setNumTestsPerEvictionRun(3);
config.setTimeBetweenEvictionRunsMillis(Duration.ofMinutes(5).toMillis());
config.setMinIdle(5);
config.setMaxTotal(20);
config.setTestOnBorrow(false);
config.setTestWhileIdle(true);
```
根据上面的配置我们可知：
1. 每次最多检测3个连接
2. 每隔5分钟检测一次
3. 最小空闲连接数为5
4. 最大连接数为20
5. 关闭获取连接时检查连接可用
6. 开启空闲连接检测
7. *最大连接空闲数为8（这边没有明确设置，是Pool的默认值）*

带着这些已知的情况，我们去询问了DBA Redis的keepalive的设置。  
DBA回复我们说：***5分钟***  
至此我们知道了原因，那就是Pool的设置和Redis的配置不匹配引起的。  
我们设定一个场景来推演：
1. 并发10次使用Pool操作Redis
2. 当操作完成后Pool中应该还有8个空闲连接（最大连接空闲数为8，所以这边不是10）
3. 当5分钟过后再次进行并发10次的Redis操作
3. 应该会出现5次`Unexpected end of stream`异常（5个新连接被建立，5个旧连接抛出异常）

### 为什么会出现5次异常？
&emsp;&emsp;因为根据Pool的设置，每5分钟才会检查池中的3个Redis连接是否正常，但当时池中有8个空闲的连接，也就是说还有5个连接在客户端是未知状态（8-3=5），这5个连接可能是可用的，也可能是不可用的，这取决于Redis的设置。  
&emsp;&emsp;而当下Redis设置的也是5分钟，也就是说这8个连接全是不可用的，Pool根据空闲检查机制帮我们剔除了3个，那么还有5个连接是会被直接使用的，那么就会抛出5次异常。

### 重现问题以验证推演
验证代码如下：
```java
JedisPoolConfig config = new JedisPoolConfig();
config.setNumTestsPerEvictionRun(3);
config.setTimeBetweenEvictionRunsMillis(Duration.ofMinutes(5).toMillis());
config.setMinIdle(5);
config.setMaxTotal(20);
config.setTestOnBorrow(false);
config.setTestWhileIdle(true);

JedisPool pool = new JedisPool(config, "x.x.x.x", 6379, 5000, "123456", 0);

List<Integer> connectionNumbers = Stream.of(0, 1, 2, 3, 4, 5, 6, 7, 8, 9).collect(Collectors.toList());

// 从池中获取10个连接后并一起关闭
connectionNumbers.stream().map(i -> pool.getResource()).collect(Collectors.toList())
        .forEach(Jedis::close);

System.out.println(String.format("active: %d", pool.getNumActive()));
System.out.println(String.format("idle: %d", pool.getNumIdle()));

// 等待5分钟 + 5秒钟（避免刚好卡在5分钟时间点）
Thread.sleep(Duration.ofMinutes(5).toMillis() + 5000);

System.out.println(LocalDateTime.now());
System.out.println(String.format("active: %d", pool.getNumActive()));
System.out.println(String.format("idle: %d", pool.getNumIdle()));

// 从Pool中取出10个连接，来进行redis操作
connectionNumbers.stream()
        .map(i -> pool.getResource())
        .collect(Collectors.toList())
        .forEach(resource -> {
            try {
                resource.get("key");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
```
结果如下：
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6wry9qdi7j21j20diqaz.jpg)
确实发生了5次 `Unexpected end of stream` 异常。

# 写在最后
Jedis的连接池基于Apache Common中的连接池，大多数java中的连接池都是基于Apache。  
所以该问题同样适用于常见的JDBC连接池。  
## 关于TCP
可以发现，TCP协议"一厢情愿"总会出问题，更多时候得"你知我知"才能正常的使用。  
TCP协议是真的很复杂的一个通信协议，不单单是三次握手4次挥手这么简单的内容。