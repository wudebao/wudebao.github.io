---
layout:     post
title:      Golang工程经验
subtitle:   Golang工程经验
date:       2017-12-12
author:     wudebao
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Blog
---

> 正所谓前人栽树，后人乘凉。
> 
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板
> 

# Golang工程经验
【记录于2017年底】

作为一个C/C++的开发者而言，开启Golang语言开发之路是很容易的，从语法、语义上的理解到工程开发，都能够快速熟悉起来；相比C、C++，Golang语言更简洁，更容易写出高并发的服务后台系统

转战Golang一年有余，经历了两个线上项目的洗礼，总结出一些工程经验，一个是总结出一些实战经验，一个是用来发现自我不足之处

## Golang语言简介
Go语言是谷歌推出的一种全新的编程语言，可以在不损失应用程序性能的情况下降低代码的复杂性。Go语言专门针对多处理器系统应用程序的编程进行了优化，使用Go编译的程序可以媲美C或C++代码的速度，而且更加安全、支持并行进程。


## 基于Golang的IM系统架构

我基于Golang的两个实际线上项目都是IM系统，本文基于现有线上系统做一些总结性、引导性的经验输出。

### Golang TCP长连接 & 并发

既然是IM系统，那么必然需要TCP长连接来维持，由于Golang本身的基础库和外部依赖库非常之多，我们可以简单引用基础net网络库，来建立TCP server。一般的TCP Server端的模型，可以有一个协程【或者线程】去独立执行accept，并且是for循环一直accept新的连接，如果有新连接过来，那么建立连接并且执行Connect，由于Golang里面协程的开销非常之小，因此，TCP server端还可以一个连接一个goroutine去循环读取各自连接链路上的数据并处理。当然， 这个在C++语言的TCP Server模型中，一般会通过EPoll模型来建立server端，这个是和C++的区别之处。

关于读取数据，Linux系统有recv和send函数来读取发送数据，在Golang中，自带有io库，里面封装了各种读写方法，如io.ReadFull，它会读取指定字节长度的数据

为了维护连接和用户，并且一个连接一个用户的一一对应的，需要根据连接能够找到用户，同时也需要能够根据用户找到对应的连接，那么就需要设计一个很好结构来维护。我们最初采用map来管理，但是发现Map里面的数据太大，查找的性能不高，为此，优化了数据结构，conn里面包含user，user里面包含conn，结构如下【只包括重要字段】。

```
// 一个用户对应一个连接
type User struct {
	uid                  int64
	conn                 *MsgConn
	bLogined             bool
	BKicked              bool // 被另外登陆的一方踢下线
	bPeerClosed          bool
	BHeartBeatTimeout    bool // 心跳超时
	closeOnce            sync.Once // true表示已经调用了关闭连接操作
}

type MsgConn struct {
	conn        net.Conn
	stopChan    chan bool
	closeOnce   sync.Once
	onCloseOnce sync.Once
	lastTick    time.Time // 上次接收到包时间
	remoteAddr  string // 为每个连接创建一个唯一标识符
	user        *User  // MsgConn与User一一映射
}
```

建立TCP server 代码片段如下

```
func ListenAndServe(network, address string) {
	tcpAddr, err := net.ResolveTCPAddr(network, address)
	if err != nil {
		logger.Fatalf(nil, "ResolveTcpAddr err:%v", err)
	}
	listener, err = net.ListenTCP(network, tcpAddr)
	if err != nil {
		logger.Fatalf(nil, "ListenTCP err:%v", err)
	}
	go accept()
}

func accept() {
	for {
		conn, err := listener.AcceptTCP()
		if err == nil {
			atomic.AddInt32(&currentTcpConnNum, 1)
			if currentTcpConnNum > maxTcpConnNum {
				conn.Close()
				atomic.AddInt32(&currentTcpConnNum, -1)
				continue
			}
			
			//anti-attack
          ...
          
          // 
			imconn := NewMsgConn(conn)
			imconn.Run()
		} 
	}
}


func (conn *MsgConn) Run() {

	//on connect
	conn.onConnect()

	go func() {
		tickerRecv := time.NewTicker(time.Second * time.Duration(rateStatInterval))
		for {
			select {
			case <-conn.stopChan:
				tickerRecv.Stop()
				return
			case <-tickerRecv.C:
				conn.packetsRecv = 0
			default:
			
			   // 在 conn.parseAndHandlePdu 里面通过Golang本身的io库里面提供的方法读取数据，如io.ReadFull
				conn_closed := conn.parseAndHandlePdu()
				if conn_closed {
					tickerRecv.Stop()
					return
				}
			}
		}
	}()
}

// 将 user 和 conn 一一对应起来
func (conn *MsgConn) onConnect() *User {
	user := &User{conn: conn, durationLevel: 0, startTime: time.Now(), ackWaitMsgIdSet: make(map[int64]struct{})}
	conn.user = user
	return user
}

```

TCP Server的一个特点在于一个连接一个goroutine去处理，这样的话，每个连接独立，不会相互影响阻塞，保证能够及时读取到client端的数据。如果是C、C++程序，如果一个连接一个线程的话，如果上万个或者十万个线程，那么性能会极低甚至于无法工作，cpu会全部消耗在线程之间的调度上了，因此C、C++程序无法这样玩。Golang的话，goroutine可以几十万、几百万的在一个系统中良好运行。同时对于TCP长连接而言，一个节点上的连接数要有限制策略。

#### 连接超时
每个连接需要有心跳来维持，在心跳间隔时间内没有收到，服务端要检测超时并断开连接释放资源，golang可以很方便的引用需要的数据结构，同时对变量的赋值（包括指针）非常easy

```
var timeoutMonitorTree *rbtree.Rbtree
var timeoutMonitorTreeMutex sync.Mutex
var heartBeatTimeout time.Duration //心跳超时时间, 配置了默认值360s
var loginTimeout time.Duration     //登陆超时, 配置了默认值15s

type TimeoutCheckInfo struct {
	conn    *MsgConn
	dueTime time.Time
}


func AddTimeoutCheckInfo(conn *MsgConn) {
	timeoutMonitorTreeMutex.Lock()
	timeoutMonitorTree.Insert(&TimeoutCheckInfo{conn: conn, dueTime: time.Now().Add(loginTimeout)})
	timeoutMonitorTreeMutex.Unlock()
}

如 &TimeoutCheckInfo{}，赋值一个指针对象

```


### Golang 基础数据结构
Golang中，很多基础数据都通过库来引用，我们可以方便引用我们所需要的库，通过import包含就能直接使用，如源码里面提供了sync库，里面有mutex锁，在需要锁的时候可以包含进来

常用的如list，mutex，once，singleton等都已包含在内

1. list链表结构，当我们需要类似队列的结构的时候，可以采用，针对IM系统而言，在长连接层处理的消息id的列表，可以通过list来维护，如果用户有了回应则从list里面移除，否则在超时时间到后还没有回应，则入offline处理


2. mutex锁，当需要并发读写某个数据的时候使用，包含互斥锁和读写锁
    ```
    var ackWaitListMutex sync.RWMutex
    var ackWaitListMutex sync.Mutex
    ```

3. once表示任何时刻都只会调用一次，一般的用法是初始化实例的时候使用，代码片段如下
    
    ```
    var initRedisOnce sync.Once

    func GetRedisCluster(name string) (*redis.Cluster, error) {
    	initRedisOnce.Do(setupRedis)
    	if redisClient, inMap := redisClusterMap[name]; inMap {
    		return redisClient, nil
    	} else {
    	}
    }

    func setupRedis() {
    	redisClusterMap = make(map[string]*redis.Cluster)
    	commonsOpts := []redis.Option{
    		redis.ConnectionTimeout(conf.RedisConnTimeout),
    		redis.ReadTimeout(conf.RedisReadTimeout),
    		redis.WriteTimeout(conf.RedisWriteTimeout),
    		redis.IdleTimeout(conf.RedisIdleTimeout),
    		redis.MaxActiveConnections(conf.RedisMaxConn),
    		redis.MaxIdleConnections(conf.RedisMaxIdle),
    		}),
    		...
    	}
    }
    ```
    这样我们可以在任何需要的地方调用GetRedisCluster，并且不用担心实例会被初始化多次，once会保证一定只执行一次 

4. singleton单例模式，这个在C++里面是一个常用的模式，一般需要开发者自己通过类来实现，类的定义决定单例模式设计的好坏；在Golang中，已经有成熟的库实现了，开发者无须重复造轮子，关于什么时候该使用单例模式请自行Google。一个简单的例子如下
    
    ```
        import 	"github.com/dropbox/godropbox/singleton"
        
        var SingleMsgProxyService = singleton.NewSingleton(func() (interface{}, error) {
    	cluster, _ := cache.GetRedisCluster("singlecache")
    	return &singleMsgProxy{
    		Cluster:  cluster,
    		MsgModel: msg.MsgModelImpl,
    	}, nil
    })

    ```

### Golang interface 接口
如果说goroutine和channel是Go并发的两大基石，那么接口interface是Go语言编程中数据类型的关键。在Go语言的实际编程中，几乎所有的数据结构都围绕接口展开，接口是Go语言中所有数据结构的核心。

#### interface - 泛型编程
严格来说，在 Golang 中并不支持泛型编程。在 C++ 等高级语言中使用泛型编程非常的简单，所以泛型编程一直是 Golang 诟病最多的地方。但是使用 interface 我们可以实现泛型编程，如下是一个参考示例

```
package sort

// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package.  The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}

...

// Sort sorts data.
// It makes one call to data.Len to determine n, and O(n*log(n)) calls to
// data.Less and data.Swap. The sort is not guaranteed to be stable.
func Sort(data Interface) {
    // Switch to heapsort if depth of 2*ceil(lg(n+1)) is reached.
    n := data.Len()
    maxDepth := 0
    for i := n; i > 0; i >>= 1 {
        maxDepth++
    }
    maxDepth *= 2
    quickSort(data, 0, n, maxDepth)
}

```

Sort 函数的形参是一个 interface，包含了三个方法：Len()，Less(i,j int)，Swap(i, j int)。使用的时候不管数组的元素类型是什么类型（int, float, string…），只要我们实现了这三个方法就可以使用 Sort 函数，这样就实现了“泛型编程”。

这种方式，我在自己的项目里面也有实际应用过，具体案例就是对消息排序。

下面给一个具体示例，代码能够说明一切，一看就懂：

```
type Person struct {
Name string
Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// ByAge implements sort.Interface for []Person based on
// the Age field.
type ByAge []Person //自定义

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
    people := []Person{
        {"Bob", 31},
        {"John", 42},
        {"Michael", 17},
        {"Jenny", 26},
    }

    fmt.Println(people)
    sort.Sort(ByAge(people))
    fmt.Println(people)
}

```
 
 
#### interface - 隐藏具体实现
隐藏具体实现，这个很好理解。比如我设计一个函数给你返回一个 interface，那么你只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。

例如我们常用的context包，就是这样的，context 最先由 google 提供，现在已经纳入了标准库，而且在原有 context 的基础上增加了：cancelCtx，timerCtx，valueCtx。   
  
如果函数参数是interface或者返回值是interface，这样就可以接受任何类型的参数


    
### 基于Golang的model service 模型【类MVC模型】
在一个项目工程中，为了使得代码更优雅，需要抽象出一些模型出来，同时基于C++面向对象编程的思想，需要考虑到一些类、继承相关。在Golang中，没有类、继承的概念，但是我们完全可以通过struct和interface来建立我们想要的任何模型。在我们的工程中，抽象出一种我自认为是类似MVC的模型，但是不完全一样，个人觉得这个模型抽象的比较好，容易扩展，模块清晰。对于使用java和PHP编程的同学对这个模型应该是再熟悉不过了，我这边通过代码来说明下这个模型

1. 首先一个model包，通过interface来实现，包含一些基础方法，需要被外部引用者来具体实现

    ```
    package model

    // 定义一个基础model
    type MsgModel interface {
    	Persist(context context.Context, msg interface{}) bool
    	UpdateDbContent(context context.Context, msgIface interface{}) bool
    	GetList(context context.Context, uid, peerId, sinceMsgId, maxMsgId int64, count int) (interface{}, bool)]

    ```

2. 再定义一个msg包，用来具体实现model包中MsgModel模型的所有方法
    
    ```
    package msg
    
    type msgModelImpl struct{}
    
    var MsgModelImpl = msgModelImpl{}
    
    func (m msgModelImpl) Persist(context context.Context, msgIface interface{}) bool {
    	 // 具体实现
    }
    
    func (m msgModelImpl) UpdateDbContent(context context.Context, msgIface interface{}) bool {
        // 具体实现

    }
    
    func GetList(context context.Context, uid, peerId, sinceMsgId, maxMsgId int64, count int) (interface{}, bool)]{
        // 具体实现
    }
    
    ```


3. model 和 具体实现方定义并实现ok后，那么就还需要一个service来统筹管理
    
    ```
    package service
    
    // 定义一个msgService struct包含了model里面的UserModel和MsgModel两个model
    type msgService struct {
    	userModel  model.UserModel
    	msgModel   model.MsgModel
    }

    // 定义一个MsgService的变量，并初始化，这样通过MsgService，就能引用并访问model的所有方法
    var (
        MsgService = msgService{
    		userModel:      user.UserModelImpl,
    		msgModel:       msg.MsgModelImpl,
    	}
    )	
   
    ```

4. 调用访问
    
    ```
    import service
    
    service.MsgService.Persist(ctx, xxx)

    ```

总结一下，model对应MVC的M，service 对应 MVC的C， 调用访问的地方对应MVC的V

### Golang 基础资源的封装
在MVC模型的基础下，我们还需要考虑另外一点，就是基础资源的封装，服务端操作必然会和mysql、redis、memcache等交互，一些常用的底层基础资源，我们有必要进行封装，这是基础架构部门所需要承担的，也是一个好的项目工程所需要的

#### redis
redis，我们在github.com/garyburd/redigo/redis的库的基础上，做了一层封装，实现了一些更为贴合工程的机制和接口，redis cluster封装，支持分片、读写分离

```

// NewCluster creates a client-side cluster for callers. Callers use this structure to interact with Redis database
func NewCluster(config ClusterConfig, instrumentOpts *instrument.Options) *Cluster {
	cluster := new(Cluster)
	cluster.pool = make([]*client, len(config.Configs))
	masters := make([]string, 0, len(config.Configs))

	for i, sharding := range config.Configs {
		master, slaves := sharding.Master, sharding.Slaves
		masters = append(masters, master)

		masterAddr, masterDb := parseServer(master)

		cli := new(client)
		cli.master = &redisNode{
			server: master,
			Pool: func() *redis.Pool {
				pool := &redis.Pool{
					MaxIdle:     config.MaxIdle,
					IdleTimeout: config.IdleTimeout,
					Dial: func() (redis.Conn, error) {
						c, err := redis.Dial(
							"tcp",
							masterAddr,
							redis.DialDatabase(masterDb),
							redis.DialPassword(config.Password),
							redis.DialConnectTimeout(config.ConnTimeout),
							redis.DialReadTimeout(config.ReadTimeout),
							redis.DialWriteTimeout(config.WriteTimeout),
						)
						if err != nil {
							return nil, err
						}
						return c, err
					},
					TestOnBorrow: func(c redis.Conn, t time.Time) error {
						if time.Since(t) < time.Minute {
							return nil
						}
						_, err := c.Do("PING")
						return err
					},
					MaxActive: config.MaxActives,
				}

				if instrumentOpts == nil {
					return pool
				}

				return instrument.NewRedisPool(pool, instrumentOpts)
			}(),
		}

		// allow nil slaves
		if slaves != nil {
			cli.slaves = make([]*redisNode, 0)
			for _, slave := range slaves {
				addr, db := parseServer(slave)

				cli.slaves = append(cli.slaves, &redisNode{
					server: slave,
					Pool: func() *redis.Pool {
						pool := &redis.Pool{
							MaxIdle:     config.MaxIdle,
							IdleTimeout: config.IdleTimeout,
							Dial: func() (redis.Conn, error) {
								c, err := redis.Dial(
									"tcp",
									addr,
									redis.DialDatabase(db),
									redis.DialPassword(config.Password),
									redis.DialConnectTimeout(config.ConnTimeout),
									redis.DialReadTimeout(config.ReadTimeout),
									redis.DialWriteTimeout(config.WriteTimeout),
								)
								if err != nil {
									return nil, err
								}
								return c, err
							},
							TestOnBorrow: func(c redis.Conn, t time.Time) error {
								if time.Since(t) < time.Minute {
									return nil
								}
								_, err := c.Do("PING")
								return err
							},
							MaxActive: config.MaxActives,
						}

						if instrumentOpts == nil {
							return pool
						}

						return instrument.NewRedisPool(pool, instrumentOpts)
					}(),
				})
			}
		}

		// call init
		cli.init()

		cluster.pool[i] = cli
	}

	if config.Hashing == sharding.Ketama {
		cluster.sharding, _ = sharding.NewKetamaSharding(sharding.GetShardServers(masters), true, 6379)
	} else {
		cluster.sharding, _ = sharding.NewCompatSharding(sharding.GetShardServers(masters))
	}

	return cluster
}


```
总结一下：

1. 使用连接池提高性能，每次都从连接池里面取连接而不是每次都重新建立连接
2. 设置最大连接数和最大活跃连接（同一时刻能够提供的连接），设置合理的读写超时时间
3. 实现主从读写分离，提高性能，需要注意如果没有从库则只读主库
4. TestOnBorrow用来进行健康检测
5. 单独开一个goroutine协程用来定期保活【ping-pong】
6. hash分片算法的选择，一致性hash还是hash取模，hash取模在扩缩容的时候比较方便，一致性hash并没有带来明显的优势，我们公司内部统一建议采用hash取模
7. 考虑如何支持双写策略



#### memcache
memcached客户端代码封装，依赖 github.com/dropbox/godropbox/memcache, 实现其ShardManager接口，支持Connection Timeout，支持Fail Fast和Rehash


### goroutine & chann

实际开发过程中，经常会有这样场景，每个请求通过一个goroutine协程去做，如批量获取消息，但是，为了防止后端资源连接数太多等，或者防止goroutine太多，往往需要限制并发数。给出如下示例供参考

```
package main

import (
    "fmt"
    "sync"
    "time"
)

var over = make(chan bool)

const MAXConCurrency = 3

//var sem = make(chan int, 4) //控制并发任务数
var sem = make(chan bool, MAXConCurrency) //控制并发任务数

var maxCount = 6

func Worker(i int) bool {

    sem <- true
    defer func() {
        <-sem
    }()

    // 模拟出错处理
    if i == 5 {
        return false
    }
    fmt.Printf("now:%v num:%v\n", time.Now().Format("04:05"), i)
    time.Sleep(1 * time.Second)
    return true
}

func main() {
    //wg := &sync.WaitGroup{}
    var wg sync.WaitGroup
    for i := 1; i <= maxCount; i++ {
        wg.Add(1)
        fmt.Printf("for num:%v\n", i)
        go func(i int) {
            defer wg.Done()
            for x := 1; x <= 3; x++ {
                if Worker(i) {
                    break
                } else {
                    fmt.Printf("retry :%v\n", x)
                }
            }
        }(i)
    }
    wg.Wait() //等待所有goroutine退出
}
```


### goroutine & context.cancel
Golang 的 context非常强大，详细的可以参考我的另外一篇文章 [Golang Context分析](http://www.jianshu.com/p/e8cccb29481f)

这里想要说明的是，在项目工程中，我们经常会用到这样的一个场景，通过goroutine并发去处理某些批量任务，当某个条件触发的时候，这些goroutine要能够控制停止执行。如果有这样的场景，那么咱们就需要用到context的With 系列函数了，context.WithCancel生成了一个withCancel的实例以及一个cancelFuc，这个函数就是用来关闭ctxWithCancel中的 Done channel 函数。

示例代码片段如下

```
func Example(){

  // context.WithCancel 用来生成一个新的Context，可以接受cancel方法用来随时停止执行
	newCtx, cancel := context.WithCancel(context.Background())

	for peerIdVal, lastId := range lastIdMap {
		wg.Add(1)

		go func(peerId, minId int64) {
			defer wg.Done()

			msgInfo := Get(newCtx, uid, peerId, minId, count).([]*pb.MsgInfo)
			if msgInfo != nil && len(msgInfo) > 0 {
				if singleMsgCounts >= maxCount {
					cancel()  // 当条件触发，则调用cancel停止
					mutex.Unlock()
					return
				}
			}
			mutex.Unlock()
		}(peerIdVal, lastId)
	}

	wg.Wait()	
}	


func Get(ctx context.Context, uid, peerId, sinceId int64, count int) interface{} {
	for {
		select {
		// 如果收到Done的chan，则立马return
		case <-ctx.Done():
			msgs := make([]*pb.MsgInfo, 0)
			return msgs

		default:
			// 处理逻辑
		}
	}
}

```
### traceid & context
在大型项目工程中，为了更好的排查定位问题，我们需要有一定的技巧，Context上下文存在于一整条调用链路中，在服务端并发场景下，n多个请求里面，我们如何能够快速准确的找到一条请求的来龙去脉，专业用语就是指调用链路，通过调用链我们能够知道这条请求经过了哪些服务、哪些模块、哪些方法，这样可以非常方便我们定位问题

traceid就是我们抽象出来的这样一个调用链的唯一标识，再通过Context进行传递，在任何代码模块[函数、方法]里面都包含Context参数，我们就能形成一个完整的调用链。那么如何实现呢 ？在我们的工程中，有RPC模块，有HTTP模块，两个模块的请求来源肯定不一样，因此，要实现所有服务和模块的完整调用链，需要考虑http和rpc两个不同的网络请求的调用链


#### traceid的实现

```
const TraceKey = "traceId"

func NewTraceId(tag string) string {
	now := time.Now()
	return fmt.Sprintf("%d.%d.%s", now.Unix(), now.Nanosecond(), tag)
}

func GetTraceId(ctx context.Context) string {
	if ctx == nil {
		return ""
	}

    // 从Context里面取
	traceInfo := GetTraceIdFromContext(ctx)
	if traceInfo == "" {
		traceInfo = GetTraceIdFromGRPCMeta(ctx)
	}

	return traceInfo
}

func GetTraceIdFromGRPCMeta(ctx context.Context) string {
	if ctx == nil {
		return ""
	}
	if md, ok := metadata.FromIncomingContext(ctx); ok {
		if traceHeader, inMap := md[meta.TraceIdKey]; inMap {
			return traceHeader[0]
		}
	}
	if md, ok := metadata.FromOutgoingContext(ctx); ok {
		if traceHeader, inMap := md[meta.TraceIdKey]; inMap {
			return traceHeader[0]
		}
	}
	return ""
}

func GetTraceIdFromContext(ctx context.Context) string {
	if ctx == nil {
		return ""
	}
	traceId, ok := ctx.Value(TraceKey).(string)
	if !ok {
		return ""
	}
	return traceId
}

func SetTraceIdToContext(ctx context.Context, traceId string) context.Context {
	return context.WithValue(ctx, TraceKey, traceId)
}


```

#### http的traceid
对于http的服务，请求方可能是客户端，也能是其他服务端，http的入口里面就需要增加上traceid，然后打印日志的时候，将TraceID打印出来形成完整链路。如果http server采用gin来实现的话，代码片段如下，其他http server的库的实现方式类似即可

```

import	"github.com/gin-gonic/gin"

func recoveryLoggerFunc() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set(trace.TraceKey, trace.NewTraceId(c.ClientIP()))
		defer func() {
				...... func 省略实现
			}
		}()
		c.Next()
	}
}

engine := gin.New()
engine.Use(OpenTracingFunc(), httpInstrumentFunc(), recoveryLoggerFunc())


	session := engine.Group("/sessions")
	session.Use(sdkChecker)
	{
		session.POST("/recent", httpsrv.MakeHandler(RecentSessions))
	}


这样，在RecentSessions接口里面如果打印日志，就能够通过Context取到traceid

```
### access log
access log是针对http的请求来的，记录http请求的API，响应时间，ip，响应码，用来记录并可以统计服务的响应情况，当然，也有其他辅助系统如SLA来专门记录http的响应情况

Golang语言实现这个也非常简单，而且这个是个通用功能，建议可以抽象为一个基础模块，所有业务都能import后使用

```
大致格式如下：

http_log_pattern='%{2006-01-02T15:04:05.999-0700}t %a - %{Host}i "%r" %s - %T "%{X-Real-IP}i" "%{X-Forwarded-For}i" %{Content-Length}i - %{Content-Length}o %b %{CDN}i'

		"%a", "${RemoteIP}",
		"%b", "${BytesSent|-}",
		"%B", "${BytesSent|0}",
		"%H", "${Proto}",
		"%m", "${Method}",
		"%q", "${QueryString}",
		"%r", "${Method} ${RequestURI} ${Proto}",
		"%s", "${StatusCode}",
		"%t", "${ReceivedAt|02/Jan/2006:15:04:05 -0700}",
		"%U", "${URLPath}",
		"%D", "${Latency|ms}",
		"%T", "${Latency|s}",

具体实现省略

```

最终得到的日志如下：

```
2017-12-20T20:32:58.787+0800 192.168.199.15 - premsg.arim.m.com:50001 "POST /arcp/unregister HTTP/1.1" 200 - 0.035 "-" "-" 14 - - 13 -
2017-12-20T20:33:27.741+0800 192.168.199.15 - premsg.arim.m.com:50001 "POST /arcp/register HTTP/1.1" 200 - 0.104 "-" "-" 68 - - 13 -
2017-12-20T20:42:01.803+0800 192.168.199.15 - premsg.arim.m.com:50001 "POST /arcp/unregister HTTP/1.1" 200 - 0.035 "-" "-" 14 - - 13 -
```

### Golang rpc 框架

### 开关策略、降级策略
线上服务端系统，必须要有降级机制，也最好能够有开关机制。降级机制在于出现异常情况能够舍弃某部分服务保证其他主线服务正常；开关也有着同样的功效，在某些情况下打开开关，则能够执行某些功能或者说某套功能，关闭开关则执行另外一套功能或者不执行某个功能。

这不是Golang的语言特性，但是是工程项目里面必要的，在Golang项目中的具体实现代码片段如下：

```
package switches


var (
	xxxtalkSwitchManager = SwitchManager{switches: make(map[string]*Switch)}
	
   AsyncProcedure = &Switch{Name: "xxxtalk.msg.procedure.async", On: true}

	// 使能音视频
	EnableRealTimeVideo = &Switch{Name: "xxxtalk.real.time.video", On: true}

)

func init() {
	xxxtalkSwitchManager.Register(AsyncProcedure, 
	EnableRealTimeVideo)
}


// 具体实现结构和实现方法
type Switch struct {
	Name      string
	On        bool
	listeners []ChangeListener
}

func (s *Switch) TurnOn() {
	s.On = true
	s.notifyListeners()
}

func (s *Switch) notifyListeners() {
	if len(s.listeners) > 0 {
		for _, l := range s.listeners {
			l.OnChange(s.Name, s.On)
		}
	}
}

func (s *Switch) TurnOff() {
	s.On = false
	s.notifyListeners()
}

func (s *Switch) IsOn() bool {
	return s.On
}

func (s *Switch) IsOff() bool {
	return !s.On
}

func (s *Switch) AddChangeListener(l ChangeListener) {
	if l == nil {
		return
	}
	s.listeners = append(s.listeners, l)
}

type SwitchManager struct {
	switches map[string]*Switch
}

func (m SwitchManager) Register(switches ...*Switch) {
	for _, s := range switches {
		m.switches[s.Name] = s
	}
}

func (m SwitchManager) Unregister(name string) {
	delete(m.switches, name)
}

func (m SwitchManager) TurnOn(name string) (bool, error) {
	if s, ok := m.switches[name]; ok {
		s.TurnOn()
		return true, nil
	} else {
		return false, errors.New("switch " + name + " is not registered")
	}
}

func (m SwitchManager) TurnOff(name string) (bool, error) {
	if s, ok := m.switches[name]; ok {
		s.TurnOff()
		return true, nil
	} else {
		return false, errors.New("switch " + name + " is not registered")
	}
}

func (m SwitchManager) IsOn(name string) (bool, error) {
	if s, ok := m.switches[name]; ok {
		return s.IsOn(), nil
	} else {
		return false, errors.New("switch " + name + " is not registered")
	}
}

func (m SwitchManager) List() map[string]bool {
	switches := make(map[string]bool)
	for name, switcher := range m.switches {
		switches[name] = switcher.On
	}
	return switches
}

type ChangeListener interface {
	OnChange(name string, isOn bool)
}


// 这里开始调用
if switches.AsyncProcedure.IsOn() {
    // do sth
}else{
    // do other sth
}

```


### prometheus + grafana
prometheus + grafana 是业界常用的监控方案，prometheus进行数据采集，grafana进行图表展示。

Golang里面prometheus进行数据采集非常简单，有对应client库，应用程序只需暴露出http接口即可，这样，prometheus server端就可以定期采集数据，并且还可以根据这个接口来监控服务端是否异常【如挂掉的情况】。

```
import 	"github.com/prometheus/client_golang/prometheus"

engine.GET("/metrics", gin.WrapH(prometheus.Handler()))

```

这样就实现了数据采集，但是具体采集什么样的数据，数据从哪里生成的，还需要进入下一步：

```
package prometheus

import "github.com/prometheus/client_golang/prometheus"

var DefaultBuckets = []float64{10, 50, 100, 200, 500, 1000, 3000}

var MySQLHistogramVec = prometheus.NewHistogramVec(
	prometheus.HistogramOpts{
		Namespace: "demo",
		Subsystem: "xxxtalk",
		Name:      "mysql_op_milliseconds",
		Help:      "The mysql database operation duration in milliseconds",
		Buckets:   DefaultBuckets,
	},
	[]string{"db"},
)

var RedisHistogramVec = prometheus.NewHistogramVec(
	prometheus.HistogramOpts{
		Namespace: "demo",
		Subsystem: "xxxtalk",
		Name:      "redis_op_milliseconds",
		Help:      "The redis operation duration in milliseconds",
		Buckets:   DefaultBuckets,
	},
	[]string{"redis"},
)

func init() {
	prometheus.MustRegister(MySQLHistogramVec)
	prometheus.MustRegister(RedisHistogramVec)
	...
}


// 使用，在对应的位置调用prometheus接口生成数据
instanceOpts := []redis.Option{
		redis.Shards(shards...),
		redis.Password(viper.GetString(conf.RedisPrefix + name + ".password")),
		redis.ClusterName(name),
		redis.LatencyObserver(func(name string, latency time.Duration) {
			prometheus.RedisHistogramVec.WithLabelValues(name).Observe(float64(latency.Nanoseconds()) * 1e-6)
		}),
	}
	
	
```


### 捕获异常  和 错误处理

#### panic 异常
捕获异常是否有存在的必要，根据各自不同的项目自行决定，但是一般出现panic，如果没有异常，那么服务就会直接挂掉，如果能够捕获异常，那么出现panic的时候，服务不会挂掉，只是当前导致panic的某个功能，无法正常使用，个人建议还是在某些有必要的条件和入口处进行异常捕获。

常见抛出异常的情况：数组越界、空指针空对象，类型断言失败等；Golang里面捕获异常通过 defer +  recover来实现

C++有try。。。catch来进行代码片段的异常捕获，Golang里面有recover来进行异常捕获，这个是Golang语言的基本功，是一个比较简单的功能，不多说，看代码

```
func consumeSingle(kafkaMsg *sarama.ConsumerMessage) {
	var err error
	defer func() {
		if r := recover(); r != nil {
			if e, ok := r.(error); ok {
				// 异常捕获的处理
			}
		}
	}()
}

```

在请求来源入口处的函数或者某个方法里面实现这么一段代码进行捕获，这样，只要通过这个入口出现的异常都能被捕获，并打印详细日志

#### error 错误

error错误，可以自定义返回，一般工程应用中的做法，会在方法的返回值上增加一个error返回值，Golang允许每个函数返回多个返回值，增加一个error的作用在于，获取函数返回值的时候，根据error参数进行判断，如果是nil表示没有错误，正常处理，否则处理错误逻辑。这样减少代码出现异常情况

#### panic 抛出的堆栈信息排查
如果某些情况下，没有捕获异常，程序在运行过程中出现panic，一般都会有一些堆栈信息，我们如何根据这些堆栈信息快速定位并解决呢 ？

一般信息里面都会表明是哪种类似的panic，如是空指针异常还是数组越界，还是xxx；

然后会打印一堆信息出来包括出现异常的代码调用块及其文件位置，需要定位到最后的位置然后反推上去

分析示例如下

```
{"date":"2017-11-22 19:33:20.921","pid":17,"level":"ERROR","file":"recovery.go","line":16,"func":"1","msg":"panic in /Message.MessageService/Proces
s: runtime error: invalid memory address or nil pointer dereference
gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/demo/commons/interceptor.newUnaryServerRecoveryInterceptor.func1.1
        /www/jenkins_home/.jenkins/jobs/demo/jobs/demo--myProject/workspace/src/gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/demo/commons/
interceptor/recovery.go:17
runtime.call64
        /www/jenkins_home/.jenkins/tools/org.jenkinsci.plugins.golang.GolangInstallation/go1.9/go/src/runtime/asm_amd64.s:510
runtime.gopanic
        /www/jenkins_home/.jenkins/tools/org.jenkinsci.plugins.golang.GolangInstallation/go1.9/go/src/runtime/panic.go:491
runtime.panicmem
        /www/jenkins_home/.jenkins/tools/org.jenkinsci.plugins.golang.GolangInstallation/go1.9/go/src/runtime/panic.go:63
runtime.sigpanic
        /www/jenkins_home/.jenkins/tools/org.jenkinsci.plugins.golang.GolangInstallation/go1.9/go/src/runtime/signal_unix.go:367
gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/pl_stability/mtrace-middleware-go/grpc.OpenTracingClientInterceptor.func1
        /www/jenkins_home/.jenkins/jobs/demo/jobs/demo--myProject/workspace/src/gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/pl_stability/m
trace-middleware-go/grpc/client.go:49
gitlab.demo.com/demo/myProject/vendor/github.com/grpc-ecosystem/go-grpc-middleware.ChainUnaryClient.func2.1.1
        /www/jenkins_home/.jenkins/jobs/demo/jobs/demo--myProject/workspace/src/gitlab.demo.com/demo/myProject/vendor/github.com/grpc-ecosystem/go-gr
pc-middleware/chain.go:90
gitlab.demo.com/demo/myProject/vendor/github.com/grpc-ecosystem/go-grpc-middleware/retry.UnaryClientInterceptor.func1


```
                                                                                                                     

问题分析

通过报错的堆栈信息，可以看到具体错误是“runtime error: invalid memory address or nil pointer dereference”，也就是空指针异常，然后逐步定位日志，可以发现最终导致出现异常的函数在这个，如下：

```
gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/pl_stability/mtrace-middleware-go/grpc.OpenTracingClientInterceptor.func1

     /www/jenkins_home/.jenkins/jobs/demo/jobs/demo--myProject/workspace/src/gitlab.demo.com/demo/myProject/vendor/gitlab.demo.com/pl_stability/m
trace-middleware-go/grpc/client.go:49
```


一般panic，都会有上述错误日志，然后通过日志，可以追踪到具体函数，然后看到OpenTracingClientInterceptor后，是在client.go的49行，然后开始反推，通过代码可以看到，可能是trace指针为空。然后一步一步看是从哪里开始调用的

最终发现代码如下：

```
	ucConn, err := grpcclient.NewClientConn(conf.Discovery.UserCenter, newBalancer, time.Second*3, conf.Tracer)
	if err != nil {
		logger.Fatalf(nil, "init user center client connection failed: %v", err)
		return
	}
	UserCenterClient = pb.NewUserCenterServiceClient(ucConn)
	
```
	
那么开始排查，conf.Tracer是不是可能为空，在哪里初始化，初始化有没有错，然后发现这个函数是在init里面，然后conf.Tracer确实在main函数里面显示调用的，main函数里面会引用或者间接引用所有包，那么init就一定在main之前执行。

这样的话，init执行的时候，conf.Tracer还没有被赋值，因此就是nil，就会导致panic了


## 项目工程级别接口
项目中如果能够有一些调试debug接口，有一些pprof性能分析接口，有探测、健康检查接口的话，会给整个项目在线上稳定运行带来很大的作用。 除了pprof性能分析接口属于Golang特有，其他的接口在任何语言都有，这里只是表明在一个工程中，需要有这类型的接口

### 上下线接口
我们的工程是通过etcd进行服务发现和注册的，同时还提供http服务，那么就需要有个机制来上下线，这样上线过程中，如果服务本身还没有全部启动完成准备就绪，那么就暂时不要在etcd里面注册，不要上线，以免有请求过来，等到就绪后再注册；下线过程中，先从etcd里面移除，这样流量不再导入过来，然后再等待一段时间用来处理还未完成的任务

我们的做法是，start 和 stop 服务的时候，调用API接口，然后再在服务的API接口里面注册和反注册到etcd

```

    var OnlineHook = func() error {
    	return nil
    }
    
    var OfflineHook = func() error {
    	return nil
    }


   // 初始化两个函数，注册和反注册到etcd的函数
	api.OnlineHook = func() error {
		return registry.Register(conf.Discovery.RegisterAddress)
	}

	api.OfflineHook = func() error {
		return registry.Deregister()
	}
	
	
	// 设置在线的函数里面分别调用上述两个函数，用来上下线
    func SetOnline(isOnline bool) (err error) {
    	if conf.Discovery.RegisterEnabled {
    		if !isServerOnline && isOnline {
    			err = OnlineHook()
    		} else if isServerOnline && !isOnline {
    			err = OfflineHook()
    		}
    	}
    
    	if err != nil {
    		return
    	}
    
    	isServerOnline = isOnline
    	return
    }
	
    
    SetOnline 为Http API接口调用的函数

```


### nginx 探测接口，健康检查接口
对于http的服务，一般访问都通过域名访问，nginx配置代理，这样保证服务可以随意扩缩容，但是nginx既然配置了代码，后端节点的情况，就必须要能够有接口可以探测，这样才能保证流量导入到的节点一定的在健康运行中的节点；为此，服务必须要提供健康检测的接口，这样才能方便nginx代理能够实时更新节点。

这个接口如何实现？nginx代理一般通过http code来处理，如果返回code=200，认为节点正常，如果是非200，认为节点异常，如果连续采样多次都返回异常，那么nginx将节点下掉

如提供一个/devops/status 的接口，用来检测，接口对应的具体实现为：

```
func CheckHealth(c *gin.Context) {
   // 首先状态码设置为非200，如503
	httpStatus := http.StatusServiceUnavailable
	// 如果当前服务正常，并服务没有下线，则更新code
	if isServerOnline {
		httpStatus = http.StatusOK
	}

    // 否则返回code为503
	c.IndentedJSON(httpStatus, gin.H{
		onlineParameter: isServerOnline,
	})
}

```


### PProf性能排查接口

```

	// PProf
	profGroup := debugGroup.Group("/pprof")
	profGroup.GET("/", func(c *gin.Context) {
		pprof.Index(c.Writer, c.Request)
	})
	profGroup.GET("/goroutine", gin.WrapH(pprof.Handler("goroutine")))
	profGroup.GET("/block", gin.WrapH(pprof.Handler("block")))
	profGroup.GET("/heap", gin.WrapH(pprof.Handler("heap")))
	profGroup.GET("/threadcreate", gin.WrapH(pprof.Handler("threadcreate")))

	profGroup.GET("/cmdline", func(c *gin.Context) {
		pprof.Cmdline(c.Writer, c.Request)
	})
	profGroup.GET("/profile", func(c *gin.Context) {
		pprof.Profile(c.Writer, c.Request)
	})
	profGroup.GET("/symbol", func(c *gin.Context) {
		pprof.Symbol(c.Writer, c.Request)
	})
	profGroup.GET("/trace", func(c *gin.Context) {
		pprof.Trace(c.Writer, c.Request)
	})
```

### debug调试接口
```
	// Debug
	debugGroup := engine.Group("/debug")
	debugGroup.GET("/requests", func(c *gin.Context) {
		c.Writer.Header().Set("Content-Type", "text/html; charset=utf-8")
		trace.Render(c.Writer, c.Request, true)
	})
	debugGroup.GET("/events", func(c *gin.Context) {
		c.Writer.Header().Set("Content-Type", "text/html; charset=utf-8")
		trace.RenderEvents(c.Writer, c.Request, true)
	})
```

### 开关【降级】实时调整接口
前面有讲到过，在代码里面需要有开关和降级机制，并讲了实现示例，那么如果需要能够实时改变开关状态，并且实时生效，我们就可以提供一下http的API接口，供运维人员或者开发人员使用。

```
	// Switch
	console := engine.Group("/switch")
	{
		console.GET("/list", httpsrv.MakeHandler(ListSwitches))
		console.GET("/status", httpsrv.MakeHandler(CheckSwitchStatus))
		console.POST("/turnOn", httpsrv.MakeHandler(TurnSwitchOn))
		console.POST("/turnOff", httpsrv.MakeHandler(TurnSwitchOff))
	}

```

## go test 单元测试用例
单元测试用例是必须，是自测的一个必要手段，Golang里面单元测试非常简单，import testing 包，然后执行go test，就能够测试某个模块代码

如，在某个user文件夹下有个user包，包文件为user.go，里面有个Func UpdateThemesCounts，如果想要进行test，那么在同级目录下，建立一个user_test.go的文件，包含testing包，编写test用例，然后调用go test即可

一般的规范有：

* 每个测试函数必须导入testing包
* 测试函数的名字必须以Test开头，可选的后缀名必须以大写字母开头
* 将测试文件和源码放在相同目录下,并将名字命名为{source_filename}_test.go
* 通常情况下，将测试文件和源码放在同一个包内。

如下：

```

// user.go
func  UpdateThemesCounts(ctx context.Context, themes []int, count int) error {
	redisClient := model.GetRedisClusterForTheme(ctx)
	key := themeKeyPattern
	for _, theme := range themes {
		if redisClient == nil {
			return errors.New("now redis client")
		}

		total, err := redisClient.HIncrBy(ctx, key, theme, count)
		if err != nil {
			logger.Errorf(ctx, "add key:%v for theme:%v count:%v failed:%v", key, theme, count, err)
			return err
		} else {
			logger.Infof(ctx, "now key:%v theme:%v total:%v", key, theme, total)
		}
	}

	return nil
}


//user_test.go
package user

import (
	"fmt"
	"testing"
	"Golang.org/x/net/context"
)

func TestUpdateThemeCount(t *testing.T) {
	ctx := context.Background()
	theme := 1
	count := 123
	total, err := UpdateThemeCount(ctx, theme, count)
	fmt.Printf("update theme:%v counts:%v err:%v \n", theme, total, err)
}

在此目录下执行 go test即可出结果

```

### 测试单个文件 or 测试单个包
通常，一个包里面会有多个方法，多个文件，因此也有多个test用例，假如我们只想测试某一个方法的时候，那么我们需要指定某个文件的某个方案

如下：

```
demo@demodeMacBook-Pro-4:~/Documents/work_demo/goDev/Applications/src/gitlab.demo.com/avatar/app_server/service/centralhub$tree .
.
├── msghub.go
├── msghub_test.go
├── pushhub.go
├── rtvhub.go
├── rtvhub_test.go
├── userhub.go
└── userhub_test.go

0 directories, 7 files

```
总共有7个文件，其中有三个test文件，假如我们只想要测试rtvhub.go里面的某个方法，如果直接运行go test，就会测试所有test.go文件了。

因此我们需要在go test 后面再指定我们需要测试的test.go 文件和 它的源文件，如下：

```
go test -v msghub.go  msghub_test.go 

```


### 测试单个文件下的单个方法
在测试单个文件之下，假如我们单个文件下，有多个方法，我们还想只是测试单个文件下的单个方法，要如何实现？我们需要再在此基础上，用 -run 参数指定具体方法或者使用正则表达式。

假如test文件如下：

```
package centralhub

import (
	"context"
	"testing"
)

func TestSendTimerInviteToServer(t *testing.T) {
	ctx := context.Background()

	err := sendTimerInviteToServer(ctx, 1461410596, 1561445452, 2)
	if err != nil {
		t.Errorf("send to server friendship build failed. %v", err)
	}
}

func TestSendTimerInvite(t *testing.T) {
	ctx := context.Background()
	err := sendTimerInvite(ctx, "test", 1461410596, 1561445452)
	if err != nil {
		t.Errorf("send timeinvite to client failed:%v", err)
	}
}
```

```
go test -v msghub.go  msghub_test.go -run TestSendTimerInvite

go test -v msghub.go  msghub_test.go -run "SendTimerInvite"
```


### 测试所有方法

指定目录即可
go test

### 测试覆盖度

go test工具给我们提供了测试覆盖度的参数，

go test -v -cover

go test -cover -coverprofile=cover.out -covermode=count

go tool cover -func=cover.out


## goalng GC 、编译运行
服务端开发者如果在mac上开发，那么Golang工程的代码可以直接在mac上编译运行，然后如果需要部署在Linux系统的时候，在编译参数里面指定GOOS即可，这样可以本地调试ok后再部署到Linux服务器。

如果要部署到Linux服务，编译参数的指定为

```
ldflags="
  -X ${repo}/version.version=${version}
  -X ${repo}/version.branch=${branch}
  -X ${repo}/version.goVersion=${go_version}
  -X ${repo}/version.buildTime=${build_time}
  -X ${repo}/version.buildUser=${build_user}
"

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "${ldflags}" -o $binary_dir/$binary_name ${repo}/
```

对于GC，我们要收集起来，记录到日志文件中，这样方便后续排查和定位，启动的时候指定一下即可执行gc，收集gc日志可以重定向

```
    export GIN_MODE=release
    GODEBUG=gctrace=1 $SERVER_ENTRY 1>/dev/null 2>$LOGDIR/gc.log.`date "+%Y%m%d%H%M%S"` &

```

## Golang包管理 目录代码管理
### 目录代码管理

整个项目包括两大类，一个是自己编写的代码模块，一个是依赖的代码，依赖包需要有进行包管理，自己的编写的代码工程需要有一个合适的目录进行管理
main.go ：入口
doc ： 文档
conf ： 配置相关
ops ： 运维操作相关【http接口】
api ： API接口【http交互接口】
daemon ： 后台daemon相关
model ： model模块，操作底层资源
service ： model的service
grpcclient ： rpc client
registry ： etcd 注册
processor ： 异步kafka消费

```
.
├── Godeps
├── README.md
├── api
├── build.sh
├── conf
├── daemon
├── dist
├── doc
├── grpcclient
├── logs
├── main.go
├── misc
├── model
├── ops
├── processor
├── push
├── registry
├── service
├── statisproxy
├── tools
├── vendor
└── version
```

### 包管理
go允许import不同代码库的代码，例如github.com, golang.org等等；对于需要import的代码，可以使用 go get 命令取下来放到GOPATH对应的目录中去。

对于go来说，其实并不care你的代码是内部还是外部的，总之都在GOPATH里，任何import包的路径都是从GOPATH开始的；唯一的区别，就是内部依赖的包是开发者自己写的，外部依赖的包是go get下来的。

依赖GOPATH来解决go import有个很严重的问题：如果项目依赖的包做了修改，或者干脆删掉了，会影响到其他现有的项目。为了解决这个问题，go在1.5版本引入了vendor属性(默认关闭，需要设置go环境变量GO15VENDOREXPERIMENT=1)，并在1.6版本之后都默认开启了vendor属性。 这样的话，所有的依赖包都在项目工程的vendor中了，每个项目都有各自的vendor，互不影响；但是vendor里面的包没有版本信息，不方便进行版本管理。

目前市场上常用的包管理工具主要有godep、glide、dep

#### godep
godep的使用者众多，如docker，kubernetes， coreos等go项目很多都是使用godep来管理其依赖，当然原因可能是早期也没的工具可选，早期我们也是使用godep进行包管理。

使用比较简单，godep save；godep restore；godep update;

但是后面随着我们使用和项目的进一步加强，我们发现godep有诸多痛点，目前已经逐步开始弃用godep，新项目都开始采用dep进行管理了。

godep的痛点:

- godep如果遇到依赖项目里有vendor的时候就可能会导致编译不过，vendor下再嵌套vendor，就会导致编译的时候出现版本不一致的错误，会提示某个方法接口不对，全部放在当前项目的vendor下

- godep锁定版本太麻烦了，在项目进一步发展过程中，我们依赖的项目（包）可能是早期的，后面由于升级更新，某些API接口可能有变；但是我们项目如果已经上线稳定运行，我们不想用新版，那么就需要锁定某个特定版本。但是这个对于godep而言，操作着实不方便。

- godep的时候，经常会有一些包需要特定版本，然后包依赖编译不过，尤其是在多人协作的时候，本地gopath和vendor下的版本不一样，然后本地gopath和别人的gopath的版本不一样，导致编译会遇到各种依赖导致的问题


#### glide
glide也是在vendor之后出来的。glide的依赖包信息在glide.yaml和glide.lock中，前者记录了所有依赖的包，后者记录了依赖包的版本信息

glide create  # 创建glide工程，生成glide.yaml
glide install # 生成glide.lock，并拷贝依赖包
glide update  # 更新依赖包信息，更新glide.lock

因为glide官方说我们不更新功能了，只bugfix，请大家开始使用dep吧，所以鉴于此，我们在选择中就放弃了。同时，glide如果遇到依赖项目里有vendor的时候就直接跪了，dep的话，就会滤掉，不会再vendor下出现嵌套的，全部放在当前项目的vendor下

#### dep
golang官方出品，dep最近的版本已经做好了从其他依赖工具的vendor迁移过来的功能，功能很强大，是我们目前的最佳选择。不过目前还没有release1.0 ，但是已经可以用在生成环境中，对于新项目，我建议采用dep进行管理，不会有历史问题，而且当新项目上线的时候，dep也会进一步优化并且可能先于你的项目上线。

dep默认从github上拉取最新代码，如果想优先使用本地gopath，那么在3.x版本的dep需要显式参数注明，如下

```
dep init -gopath -v
```

初始化项目的时候，通过dep init进行初始化，这个操作会默认从github或者其他外网上拉取代码，并且dep会自己先猜测所需要的版本，一般默认是最新的。这里需要说明的是，如果某个项目没有tag，那么dep会从默认主干（一般是master）分支上拉取；如果有tag，那么dep会拉取最新的tag。于此同时，init操作会生成两个文件，一个是Gopkg.toml，一个Gopkg.lock

Gopkg.toml 用来做版本管理，可以手动修改，当我们需要指定某个特定版本的时候，需要手动修改里面的version。

Gopkg.lock 用来锁定工程版本和数据，不能修改，是自动生成的，会将toml里面的版本hash后再记录，如果一个项目引用了多个包，也将作为一个整体来管理

如果需要更新 dep ensure update


#### 总结
- godep是最初使用最多的，能够满足大部分需求，也比较稳定，但是有一些不太好的体验；

- glide 有版本管理，相对强大，但是官方表示不再进行开发； 

- **dep是官方出品，目前没有release，功能同样强大，是目前最佳选择；**

[看官方的对比](https://github.com/golang/go/wiki/PackageManagementTools)


## Golang容易出现的问题
### 包引用缺少导致panic
go vendor 缺失导致import多次导致panic

本工程下没有vendor目录，然而，引入了这个包“gitlab.demo.com/demo/myProject/model/impl/hash”， 这个myProject包里面包含了vendor目录。

这样，编译此工程的时候，会导致一部分import是从myProject下的vendor，另一部分是从gopath，这样就会出现一个包被两种不同方式import，导致出现重复注册而panic


### 并发 导致 panic 
fatal error: concurrent map read and map write

并发编程中最容易出现资源竞争，以前玩C++的时候，资源出现竞争只会导致数据异常，不会导致程序异常panic，Golang里面会直接抛错，这个是比较好的做法，因为异常数据最终导致用户的数据异常，影响很大，甚至无法恢复，直接抛错后交给开发者去修复代码bug，一般在测试过程中或者代码review过程中就能够发现并发问题。

并发的处理方案有二：

1. 通过chann 串行处理

2. 通过加锁控制


### 相互依赖引用导致编译不过
Golang不允许包直接相互import，会导致编译不过。但是有个项目里面，A同学负责A模块，B同学负责B模块，由于产品需求导致，A模块要调用B模块中提供的方法，B模块要调用A模块中提供的方法，这样就导致了相互引用了 

我们的解决方案是： 将其中一个相互引用的模块中的方法提炼出来，独立为另外一个模块，也就是另外一个包，这样就不至于相互引用


### Golang json类型转换异常
Golang进行json转换的时候，常用做法是一个定义struct，成员变量使用tag标签，然后通过自带的json包进行处理，容易出现的问题主要有：

1. 成员变量的首字母没有大写，导致json后生成不了对应字段
2. json string的类型不对，导致json Unmarshal 的时候抛错


## Golang 总结

golang使用一年多以来，个人认为golang有如下优点：

- 学习入门快；让开发者开发更为简洁
- 不用关心内存分配和释放，gc会帮我们处理；
- 效率性能高；
- 不用自己去实现一些基础数据结构，官方或者开源库可以直接import引用； 
- struct 和 interface 可以实现类、继承等面向对象的操作模式；
- 初始化和赋值变量简洁；
- 并发编程goroutine非常容易，结合chann可以很好的实现；
- Context能够自我控制开始、停止，传递上下文信息


