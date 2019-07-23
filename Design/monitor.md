#### 系统设计图

![](./new.jpg)

#### 组件说明

##### [Whisper](https://github.com/graphite-project/whisper)

whisper本质上是一个RRD，负责指标数据持久化存储，指标存储策略由配置文件storage-schemas.conf指定，基本格式为:

```tex
[name]
pattern = regex
retentions = timePerPoint:timeToStore, timePerPoint:timeToStore, ...
```

name用于标识一个section配置，pattern用于说明指标名称匹配规则，retentions用于说明指标存储策略，例如：

```tex
[carbon]
pattern = ^carbon\.
retentions = 60s:90d,15m:120d
```

上述配置表明，以carbon.开头的指标存储策略为最近90天每60s保存一个数据点、最近120天以每15分钟保存一个数据点。

根据retentions规则，whisper会计算出对应指标占用存储空间大小，比如60s:90d表明需要$\frac{90 * 24 * 60 * 60} {60}$个数据点，每个数据点分配的大小是已知的，所以Whisper数据文件占用存储空间是固定的。

存储聚合，指高精度数据集归档于低精度数据集。

以上述例子中retentions配置为例，每60s一个数据点分辨率是高于每15分钟一个数据点，15m一个数据点的采集是完全可以根据60s一个数据点进行归档（15 * 60刚好能整出60），所以Whisper中若出现retentions数据分辨率多于一个，是需要指定归档聚合策略，其由storage-aggregation.conf文件指定，格式为:

```tex
[name]
pattern = <regex>
xFilesFactor = <float between 0 and 1>
aggregationMethod = <average|sum|last|max|min>
```

pattern指定指标名称模式，xFilesFactor用于指定归档因子，以上述配置为例，15个60s数据点会归档到一个15m数据点，若$$xFilesFactor=0.8$$，则意味着一个段中至少存在$$15*0.8=12$$个数据点才会运用聚合函数归档，否则忽略。这也间接说明了retentions配置时的约束条件，以$$retentions=s_{1}:t_{1},s_{2}:t_{2}$$为例（$$s_{i}、t_{i}$$以秒为单位、$s_{i}$升序排列）：

- s2必须整除s1
- t2大于t1

##### [StatsD](https://github.com/statsd/statsd)

StatsD根据文本协议收集指标，以一定时间间隔聚合指标后，上传给对应后端，支持的后端有graphite、influxdb等。

StatsD支持4种指标类型：counter、timer、gauge、set。

文本协议格式为:

```tex
<bucket>:<value>|<type>[|@sample_rate]
```

简要说明：

- bucket: bucket是一个metric的标识，可以看成一个metric的变量
- value: metric的值，通常是数值
- type: metric类型，当前支持的指标类型为c(counter)、g(gauge)、s(set)、ms(timer)
- sample_rate: 采样率，仅支持counter、timer

###### gauge

```tex
age:10|g    // age 为 10
age:+1|g    // age 为 10 + 1 = 11
age:-1|g    // age为 11 - 1 = 10
age:5|g     // age为5,替代前一个值
```

gauge是任意的一维标量值，flush时其值不会清零，StatsD会将一个flush区间内最后一个值发送到后端，典型gauge类型指标如进程占用内存大小、CPU使用率、连接数等。

###### counter

```tex
user.logins:10|c        // user.logins + 10
user.logins:-1|c        // user.logins - 1
user.logins:10|c|@0.1   // user.logins + 100
                        // users.logins = 10-1+100=109
```

counter会将一个flush区间内的指标值累加，flush过后重置为0，所以counter更适合统计最近的变化值。

采样率是在客户端决定当前数据点是否上报，StatsD会根据采样率计算出原始值。

###### set

记录flush期间不重复的值

```tex
request:1|s  // user 1
request:2|s  // user1 user2
request:1|s  // user1 user2
```

若上述指标在同一个flush窗口，则Whisper中记录该时间点的值为2。

###### timer

timer会记录一个flush区间内的平均值（mean）、最大值（upper）、最小值（lower）、累加值（sum）、平方和（sum_squares）、个数（count）以及部分百分值，需要记录的百分值通过配置选项config.percentThreshold指定，默认值为90，即需要统计P90指标项，允许配置多个百分比点。

StatsD处理$x\%$百分比时，会通过原始数据点 * $x\%$得到统计数据点，据此计算对应的百分比指标。

举例来说，在一个flush期间的数据点为:

```tex
1,3,5,7,13,9,11,2,4,8
```

则P90定义为原始数据集排序后，取前$10 * 0.9=9$个数据点（四舍五入）：

 ```tex
1,3,5,7,9,11,2,4,8
 ```

各个指标项语义（如：mean、lower、sum等）与统计学中的语义一致。

#### 使用注意点

1. Whisper保留策略配置时，高分辨率时间间隔需要能被低分辨率时间间隔整除。

2. StatsD配置项中flushInterval应取值Whisper中最小的时间间隔，这与Whisper采用同一时间段内覆盖写有关。

3. StatsD发送指标到Graphite，会根据指标类型做相应的指标名称变更，对应名称转换表:

   | 数据类型 | 业务定义指标名称 | Whisper中对应指标名称 |
   | :------: | :--------------: | :-------------------: |
   | counter  |   metric_name    |   metric_name.count   |
   |  gauge   |   metric_name    |      metric_name      |
   |   set    |   metric_name    |   metric_name.count   |
   |  timer   |   metric_name    |     metric_name.*     |

#### 参考资料

[StatsD Metric Types](https://github.com/statsd/statsd/blob/master/docs/metric_types.md)

[Metric namespacing](https://github.com/statsd/statsd/blob/master/docs/namespacing.md)

[StatsD](https://github.com/statsd/statsd/wiki)

