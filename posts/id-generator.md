---
title: '分布式ID生成器'
date: '2022-05-13'
---

# 分布式ID生成器
## 分布式ID的特点
- 全局唯一性：不能出现有重复的 ID 标识，这是基本要求。
- 递增性：确保生成的 ID 对于用户或业务是递增的。
- 高可用性：确保任何时候都能生成正确的D。
- 高性能性：在高并发的环境下依然表现良好。

不仅仅是用于用户 ID ,实际互联网中有很多场景需要能够生成类似MySQL自增 ID 这样不断增大，同时又不会重复的 ID 。以支持业务中的高并发场景。

比较典型的场景有：电商促销时短时间内会有大量的订单诵入到系统，比如每秒10W+; 明星出轨时微博短时间内会产生大量的相关微博转发和评论消息。在这些业务场景下将数据插入数据库之前，我们需要给这些订单和消息先分配一个唯一ID,然后再保存到数据库中。对这个id的要求是希望其中能带有一些时间信息，这样即使我们后端的系统对消息进行了分库分表，也能够以时间顺序对这些消息进行排序。

雪花算法
### snowflake算法介绍
雪花算法，它是Twitter开源的由64位整数组成分布式ID,性能较高，并且在单机上递增。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64b3eb6ac2ab4db4a311ca94c6ecb8a6~tplv-k3u1fbpfcp-watermark.image?)

1. 第一位占用1bt,其值始终是0，没有实际作用。
2. 时间戳占用41bt,单位为毫秒，总共可以容纳约69年的时间。当然，我们的时间毫秒计数不会真的从1970年开始记，那样我们的系统跑到`2039/9/7 23：47：35`就不能用了，所以这里的时间戳只是相对于某个时间的增量，比如我们的系统上线是`2022-05-01`，那么我们完全可以把这个timestamp当作是从`2022-05-01 00：00：00.000`的偏移量。
3. 工作机器 id 占用 `10bit`,其中高位5bit是数据中心D,低位5bit是工作节点ID,最多可以容纳1024个节点。
4. 序列号占用12bt,用来记录同毫秒内产生的不同d。每个节点每毫秒0开始不断累加，最多可以累加到4095，同一毫秒一共可以产生4096个1D。
SnowFlake算法在同一毫秒内最多可以生成多少个全局唯一ID呢？
同一毫秒的1D数量=1024X4096=4194304I

## snowflake 的 Go 实现
### bwmarrin/snowflake 

[bwmarrin/snowflake: A simple to use Go (golang) package to generate or parse Twitter snowflake IDs (github.com)](https://github.com/bwmarrin/snowflake)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe907cab4d2b46c1aa2c9228a3cb8388~tplv-k3u1fbpfcp-watermark.image?)

bwmarrin/snowflake 是一个轻量化的snowflake的Go实现。

实现示例：
```go
package main

import (
   "fmt"
   "time"

   "github.com/bwmarrin/snowflake"
)

var node *snowflake.Node

func Init(startTime string, machineID int64) (err error) {
   var st time.Time
   st, err = time.Parse("2006-01-02", startTime)
   if err != nil {
      return
   }
   snowflake.Epoch = st.UnixNano() / 1000000
   node, err = snowflake.NewNode(machineID)
   return
}

func GenID() int64 {
   return node.Generate().Int64()
}

func main() {
   if err := Init("2022-05-03", 1); err != nil {
      fmt.Printf("init failed, err:%v\n", err)
      return
   }
   id := GenID()
   fmt.Println(id)
}
```

### sony/sonyflake

[sony/sonyflake: A distributed unique ID generator inspired by Twitter's Snowflake (github.com)](https://github.com/sony/sonyflake)

sonyflake，是Sony公司的一个开源项目，基本思路和snowflake:差不多，不过位分配上稍有不同：
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfcface6498747e6a16135b72299dd35~tplv-k3u1fbpfcp-watermark.image?)

这里的时间只用了39个bit,但时间的单位变成了10ms,所以理论上比41位表示的时间还要久(174年)。
Sequence ID和之前的定义一致，Machine ID其实就是节点id。sonyflake库有以下配置参数：

```go
type Settings struct {
    StartTime time.Time
    MachineID func() (uint16, error)
    CheckMachineID func(uint16) bool
}
```

简单的实例：
```go
package main

import (
   "fmt"
   "time"

   "github.com/sony/sonyflake"
)

var (
   sonyFlake     *sonyflake.Sonyflake // 类似于node
   sonyMachineID uint16
)

func getMachineID() (uint16, error) {
   return sonyMachineID, nil
}

func Init(startTime string, machineID uint16) (err error) {
   sonyMachineID = machineID
   var st time.Time
   st, err = time.Parse("2006-01-02", startTime)
   if err != nil {
      return err
   }
   settings := sonyflake.Settings{
      StartTime: st,
      MachineID: getMachineID,
   }
   sonyFlake = sonyflake.NewSonyflake(settings)
   return
}

// GenID 生成id
func GenID() (id uint64, err error) {
   if err != nil {
      fmt.Errorf("sonyflake init failed, err:%v\n", err)
      return
   }
   id, err = sonyFlake.NextID()
   return
}

func main() {
   if err := Init("2022-05-03", 1); err != nil {
      fmt.Printf("Init failed, err:%v\n", err)
      return
   }
   id, _ := GenID()
   fmt.Println(id)
}
```

