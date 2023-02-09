---
layout: post
title: "Ceph CRUSH Map简介"
date:   2022-09-07
tags: [ceph]
comments: true
author: xakdwch
---

# **Overview**

CRUSH是ceph的核心设计之一，CRUSH算法通过简单计算就能确定数据的存储位置。因此ceph客户端无需经过传统查表的方式来获取数据的索引，进而根据索引来读写数据，只需通过crush算法计算后直接和对应的OSD交互进行数据读写。这样，ceph就避免了查表这种传统中心化架构存在的单点故障、性能瓶颈以及不易扩展的缺陷。

CRUSH map是ceph集群物理拓扑的抽象，CRUSH算法通过CRUSH map中的ceph集群拓扑结构、副本策略以及故障域等信息，将数据伪随机地分布到集群的各个OSD上。

因此，读懂CRUSH map也有助于我们理解CRUSH算法。

# **CRUSH map含义**

结合实际的ceph环境，讲解CRUSH map中配置项的含义和作用。

## **CRUSH map树状层级结构**

现有ceph集群的树状层级结构如下：

![img](https://github.com/xakdwch/xakdwch.github.io/blob/master/images/osd_tree.jpg)

这是一个3节点，6 osd的ceph集群，集群为3副本模式，故障域为host。树状结构的根节点为root，叶子节点为osd，中间节点为host。

## **获取集群CRUSH map**

```shell
# 获取集群的CRUSH map
ceph osd getcrushmap -o {compiled-crushmap-filename}
例：ceph osd getcrushmap -o crush.bin
# 反编译CRUSH map，得到明文内容
crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
例：crushtool -d crush.bin -o crush.txt
```

打开crush.txt内容如下：

```shell
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root

# buckets
host host042244 {
    id -3       # do not change unnecessarily
    id -4 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.0 weight 0.195
    item osd.3 weight 0.488
}
host host042245 {
    id -5       # do not change unnecessarily
    id -6 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.1 weight 0.195
    item osd.4 weight 0.488
}
host host042246 {
    id -7       # do not change unnecessarily
    id -8 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.2 weight 0.195
    item osd.5 weight 0.488
}
root default {
    id -1       # do not change unnecessarily
    id -2 class hdd     # do not change unnecessarily
    # weight 2.051
    alg straw2
    hash 0  # rjenkins1
    item host042244 weight 0.684
    item host042245 weight 0.684
    item host042246 weight 0.684
}

# rules
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}

# end crush map
```

## **CRUSH map的组成**

CRUSH map主要有以下几部分组成：

1. tunables：tunables参数主要用来修正一些旧bug、优化算法、以及向后兼容老版本
2. devices：代表各个存储数据的osd，是CRUSH树状结构的叶子节点
3. types：bucket的类型，可以自定义，编号为正整数
4. buckets：CRUSH树状结构的所有中间节点就叫bucket，bucket可以是一些device的集合，也可以是诸如（host、rack、room等）的集合，根节点root是整个集群的入口
5. rules：定义了数据在集群中的分布策略，crush算法的每一步都按照规则来执行

### **tunables参数**

```shell
# 为做向后兼容应保持为0
 tunable choose_local_tries 0
# 为做向后兼容应保持为0
 tunable choose_local_fallback_tries 0
# 选择bucket的最大重试次数，如果重试50次crush算法还未选择出期望的bucket，可以调整此参数
 tunable choose_total_tries 50
# 为做向后兼容应保持为1
 tunable chooseleaf_descend_once 1
# 修复旧bug，为做向后兼容应保持为1
 tunable chooseleaf_vary_r 1
# 避免不必要的pg迁移，为做向后兼容应保持为1
 tunable chooseleaf_stable 1
# straw算法版本，为做向后兼容应保持为1
 tunable straw_calc_version 1
# 允许使用的bucket选择算法，通过位运算计算得出的值
 tunable allowed_bucket_algs 54
```

bucket算法有以下几种：

```shell
enum crush_algorithm {
    CRUSH_BUCKET_UNIFORM = 1,   //uniform算法
    CRUSH_BUCKET_LIST = 2,      //list算法
    CRUSH_BUCKET_TREE = 3,      //tree算法
    CRUSH_BUCKET_STRAW = 4,     //straw算法
    CRUSH_BUCKET_STRAW2 = 5,    //straw2算法
};
```

最优crush map默认参数设置：

```shell
void set_optimal_crush_map(struct crush_map *map) {
  map->choose_local_tries = 0;
  map->choose_local_fallback_tries = 0;
  map->choose_total_tries = 50;
  map->chooseleaf_descend_once = 1;
  map->chooseleaf_vary_r = 1;
  map->chooseleaf_stable = 1;
  map->allowed_bucket_algs = (
    (1 << CRUSH_BUCKET_UNIFORM) |
    (1 << CRUSH_BUCKET_LIST) |
    (1 << CRUSH_BUCKET_STRAW) |
    (1 << CRUSH_BUCKET_STRAW2));
}
```

### **devices**

devices定义了集群中各个OSD daemon，每个device有一个对应的id（非负数）和name，通常osd.1中的1就是device的id。

device也可以设置对应的类别，比如hdd、ssd、nvme等磁盘介质类型，这样方便crush规则根据介质类型来选择device。

```shell
# devices格式
device {num} {osd.name} [class {class}]

# devices示例
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
```

### **types**

代表bucket的类型，bucket代表CRUSH map树状层级结构中的各node。树的每个node又对应着ceph集群的每个物理拓扑位置，每个bucket可以是低一层级的bucket集合或是一些叶子node的集合。

用户可以定义新的type，按照惯例，叶子节点必需为osd，且其type为0。

```shell
# types格式
type {num} {bucket-name}

# types示例
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root
```

### **buckets**

所有的中间节点就叫做bucket，bucket可以是一些devices的集合也可以是低一级的buckets的集合, 根节点称为root是整个集群的入口。

CRUSH算法会根据定义的CRUSH map将数据及副本分布到集群中，除了叶子节点必须代表osd外，其它层级的bucket用户可以根据自己的实际情况任意定义。

推荐用户在CRUSH map中的命名，可以反映出自己实际的拓扑结构、硬件名称等信息，越详细越接近实际就越好。好的命名可以使你更容易对集群进行运维管理，当集群osd出现硬件故障时，可以更快更准确的定位到osd对应的磁盘位置。

当定义一个bucket时，需要遵循以下语法规范：

```shell
[bucket-type] [bucket-name] {
    # 全局唯一的负数id
    id [a unique negative numeric ID]
    # 所有item的存储容量，1代表1TB，0.5代表0.5TB，依此类推
    weight [the relative capacity/capability of the item(s)]
    # bucket选择使用的crush算法类型，uniform | list | tree | straw | straw2
    alg [the bucket type: uniform | list | tree | straw | straw2 ]
    # bucket使用的hash算法
    hash [the hash type: 0 by default]
    # 低一层级的bucket名称，以及其对应的weight
    item [item-name] weight [weight]
}
```

例如，我们的ceph环境中有3个节点（对应3个bucket：host042244、host042245、host042246），每个节点上都有两块hdd硬盘（200G+500G，weight对应为0.195和0.488），host042244节点上运行osd.0和osd.3，host042245节点上运行osd.1和osd.4，host042246节点上运行osd.2和osd.5。

```shell
# buckets
host host042244 {
    id -3       # do not change unnecessarily
    id -4 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.0 weight 0.195
    item osd.3 weight 0.488
}
host host042245 {
    id -5       # do not change unnecessarily
    id -6 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.1 weight 0.195
    item osd.4 weight 0.488
}
host host042246 {
    id -7       # do not change unnecessarily
    id -8 class hdd     # do not change unnecessarily
    # weight 0.684
    alg straw2
    hash 0  # rjenkins1
    item osd.2 weight 0.195
    item osd.5 weight 0.488
}
root default {
    id -1       # do not change unnecessarily
    id -2 class hdd     # do not change unnecessarily
    # weight 2.051
    alg straw2
    hash 0  # rjenkins1
    item host042244 weight 0.684
    item host042245 weight 0.684
    item host042246 weight 0.684
}
```

### **rules**

制定了一个pool中的数据在集群中的分布策略，crush算法的每一步都按照规则来执行。

默认情况下，CRUSH map会有一个默认rule应用到所有pool。用户也可以根据自己的需求，为每个pool指定不同的rule。比如，为A业务创建的存储池指定ssd rule，为B业务创建的存储池指定hdd rule。这样A业务根据指定的rule将数据存到ssd硬盘，从而获得更好的性能；B业务对性能要求不高，根据rule将数据存到hdd硬盘，从而可以节省硬件成本。

rule的语法规范如下：

```shell
rule <rulename> {
    # 全局唯一数字id
    id [a unique whole numeric ID]
    # 副本类型，多副本或纠删码
    type [ replicated | erasure ]
    # 如果副本数小于该值，则不会采用该rule
    min_size <min-size>
    # 如果副本数大于该值，则不会采用该rule
    max_size <max-size>
    # crush规则入口，如果指定了device-class，则必须匹配符合类型的device
    step take <bucket-name> [class <device-class>]
    # 分为choose和chooseleaf两种，N代表选择的数量，bucket-type是预期的bucket类型
    step [choose|chooseleaf] [firstn|indep] <N> type <bucket-type>
    # 代表着这个规则的结束，输出执行结果
    step emit
}
```

`step [choose|chooseleaf] [firstn|indep] <N> type <bucket-type>`这条规则详细介绍如下：

choose：选择到预期数量和类型的bucket即可结束

chooseleaf：选择到预期数量和类型的bucket，并最终从这些bucket中选出叶子节点（即osd）

firstn和indep：当出现osd down掉的情况时，用于控制CRUSH的副本策略。副本池应该选择firstn，纠删码池应该选择indep，纠删码要求选择结果是有序的。

假如，一个PG的5个副本分布在OSD 1，2，3，4，5上，然后3 down了。

在firstn模式下，CRUSH算法会选择1，2，在选择3时发现其down掉了，然后接着尝试选择4，5，最后接着选择一个新的未down掉的OSD 6，也就是最终的结果变迁为：1，2，3，4，5 -> 1，2，4，5，6。

在indep模式下，CRUSH算法会选择1，2，在选择3时发现其down掉了，会再尝试选择一个新的未down掉的OSD 6，然后接着选择4，5，也就是最终的结果变迁为：1，2，3，4，5 -> 1，2，6，4，5。

N：如果N==0，选择的数量等于副本数；如果0<N<副本数，选择的数量等于N；如果N<0，选择的数量等于（副本数-N）。

我们ceph环境的rule信息如下：

```shell
# rules
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}
```

因为我们ceph环境的crush map未自定义配置，所有存储池的CRUSH规则使用默认的replicated_rule规则，pool副本数为3。`step chooseleaf firstn 0 type host`的意思就是，选择3个host类型的bucket，并最终从这3个host中分别选出1个OSD。

## **导入集群CRUSH map**

当修改了CRUSH map并想导入集群使用时，可以执行以下命令：

```shell
# 编译新的CRUSH map文件
crushtool -c {decompiled-crushmap-filename} -o {compiled-crushmap-filename}
例：crushtool -c crush.txt -o crush.bin
# 设置新的CRUSH map
ceph osd setcrushmap -i {compiled-crushmap-filename}
例：ceph osd setcrushmap -i crush.bin
```

# **案例**

这里通过案例来展示一下CRUSH map的实际操作。

## **案例一**

### **需求**

有3个节点，每个节点上有2块SSD盘和2块HDD盘，硬件配置列表如下：

| 主机名 | HDD盘    | SSD盘      |
| :----- | :------- | :--------- |
| ceph1  | 1 TB * 2 | 500 GB * 2 |
| ceph2  | 1 TB * 2 | 500 GB * 2 |
| ceph3  | 1 TB * 2 | 500 GB * 2 |

现有业务A和业务B的存储需求如下：

- 业务A对性能要求较高，将SSD作为数据盘，需创建3副本的SSD存储池
- 业务B对性能要求不高，但数据量较大，将HDD作为数据盘降低成本，需创建3副本的HDD存储池

### **CRUSH map结构**

![img](https://github.com/xakdwch/xakdwch.github.io/blob/master/images/crush1.jpg)

### **编写CRUSH map**

首先，修改devices定义osd信息：

```shell
# devices
device 0 osd.0 class ssd
device 1 osd.1 class ssd
device 2 osd.2 class ssd
device 3 osd.3 class ssd
device 4 osd.4 class ssd
device 5 osd.5 class ssd
device 6 osd.6 class hdd
device 7 osd.7 class hdd
device 8 osd.8 class hdd
device 9 osd.9 class hdd
device 10 osd.10 class hdd
device 11 osd.11 class hdd
```

添加新的ssd bucket和hdd bucket：

```shell
# buckets
host ceph1-hdd {
    id -3       # do not change unnecessarily
    id -4 class hdd     # do not change unnecessarily
    # weight 1.95319
    alg straw2
    hash 0  # rjenkins1
    item osd.6 weight 0.97659
    item osd.7 weight 0.97659
}
host ceph2-hdd {
    id -5       # do not change unnecessarily
    id -6 class hdd     # do not change unnecessarily
    # weight 1.95319
    alg straw2
    hash 0  # rjenkins1
    item osd.8 weight 0.97659
    item osd.9 weight 0.97659
}
host ceph3-hdd {
    id -7       # do not change unnecessarily
    id -8 class hdd     # do not change unnecessarily
    # weight 1.95319
    alg straw2
    hash 0  # rjenkins1
    item osd.10 weight 0.97659
    item osd.11 weight 0.97659
}
host ceph1-ssd {
    id -9 class ssd     # do not change unnecessarily
    # weight 0.97660
    alg straw2
    hash 0  # rjenkins1
    item osd.0 weight 0.48830
    item osd.1 weight 0.48830
}
host ceph2-ssd {
    id -10 class ssd        # do not change unnecessarily
    # weight 0.97660
    alg straw2
    hash 0  # rjenkins1
    item osd.2 weight 0.48830
    item osd.3 weight 0.48830
}
host ceph3-ssd {
    id -11 class ssd        # do not change unnecessarily
    # weight 0.97660
    alg straw2
    hash 0  # rjenkins1
    item osd.4 weight 0.48830
    item osd.5 weight 0.48830
}
root rep_hdd {
    id -1       # do not change unnecessarily
    id -2 class hdd     # do not change unnecessarily
    # weight 5.85957
    alg straw2
    hash 0  # rjenkins1
    item ceph1-hdd weight 1.95319
    item ceph2-hdd weight 1.95319
    item ceph3-hdd weight 1.95319
}
root rep_ssd {
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item ceph1-ssd weight 0.9766
    item ceph2-ssd weight 0.9766
    item ceph3-ssd weight 0.9766
}
```

添加新的CRUSH rules规则：

```shell
rule hdd_rule { 
    id 0
    type replicated
    min_size 1
    max_size 10
    step take rep_hdd
    step chooseleaf firstn 0 type host
    step emit

}
rule ssd_rule { 
    id 1
    type replicated
    min_size 1
    max_size 10
    step take rep_ssd
    step chooseleaf firstn 0 type host
    step emit
}
```

修改完CRUSH map之后，将新的CRUSH map文件进行编译，并导入到集群中。

### **创建存储池**

更新完CRUSH map之后，根据业务需求，创建对应的pool并关联CRUSH rule即可。

存储池的创建命令的格式如下：

```shell
ceph osd pool create <pool> [<pg_num:int>] [<pgp_num:int>] [replicated|erasure] [<erasure_code_profile>] [<rule>] [<expected_num_objects:int>] [<size:int>] [<pg_num_min:int>] [<pg_num_max:int>] [on|off|warn] [--bulk] [<target_size_bytes:int>] [<target_size_ratio:float>]
```

分别创建ssd存储池和hdd存储池，示例如下：

```shell
# 创建hdd存储池
 ceph osd pool create hdd_pool 512 512 replicated hdd_rule
# 创建ssd存储池
 ceph osd pool create ssd_pool 256 256 replicated ssd_rule
```

## **案例二**

### **需求**

有3个节点，分别位于不同的机房，硬件配置列表如下：

| 机房   | 机架   | 主机名 | 硬盘(HDD) |
| :----- | :----- | :----- | :-------- |
| room-1 | rack-a | node-1 | 1 TB * 3  |
| room-2 | rack-b | node-2 | 1 TB * 3  |
| room-3 | rack-c | node-3 | 1 TB * 3  |

出于高可用性考虑，客户将ceph集群数据的3副本分别分布到3个不同机房中，机房间专线连接。这样即使其中1个机房断电，ceph集群仍可以正常服务。

为了在发生故障时，可以快速定位问题所在，故需要制定CRUSH map，要求能够真实地反映实际的物理拓扑。

### **CRUSH map结构**

![img](https://github.com/xakdwch/xakdwch.github.io/blob/master/images/crush2.jpg)

### **编写CRUSH map**

首先，修改devices定义osd信息：

```shell
# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
device 6 osd.6 class hdd
device 7 osd.7 class hdd
device 8 osd.8 class hdd
```

添加新的bucket：

```shell
# buckets
host node1-hdd {
    id -9       # do not change unnecessarily
    id -10 class hdd        # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item osd.0 weight 0.97659
    item osd.1 weight 0.97659
    item osd.2 weight 0.97659
}
host node2-hdd {
    id -11      # do not change unnecessarily
    id -12 class hdd        # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item osd.3 weight 0.97659
    item osd.4 weight 0.97659
    item osd.5 weight 0.97659
}
host node3-hdd {
    id -13      # do not change unnecessarily
    id -14 class hdd        # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item osd.6 weight 0.97659
    item osd.7 weight 0.97659
    item osd.8 weight 0.97659
}
rack rack-a {
    id -6 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item node1-hdd weight 2.9298
}
rack rack-b {
    id -7 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item node2-hdd weight 2.9298
}
rack rack-c {
    id -8 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item node3-hdd weight 2.9298
}
room room-1 {
    id -3 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item rack-1 weight 2.9298
}
room room-2 {
    id -4 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item rack-2 weight 2.9298
}
room room-3 {
    id -5 class hdd     # do not change unnecessarily
    # weight 2.9298
    alg straw2
    hash 0  # rjenkins1
    item rack-3 weight 2.9298
}
root rep_cross {
    id -1       # do not change unnecessarily
    id -2 class hdd     # do not change unnecessarily
    # weight 8.7894
    alg straw2
    hash 0  # rjenkins1
    item room-1 weight 2.9298
    item room-2 weight 2.9298
    item room-3 weight 2.9298
}
```

添加新的CRUSH rules规则：

```shell
rule cross_rule { 
    id 0
    type replicated
    min_size 1
    max_size 10
    step take rep_cross
    step chooseleaf firstn 0 type host
    step emit

}
```

修改完CRUSH map之后，将新的CRUSH map文件进行编译，并导入到集群中。

### **创建存储池**

示例如下：

```shell
# 创建存储池crush规则采用跨机房策略
 ceph osd pool create cross_pool 512 512 replicated cross_rule
```

# **总结**

- 在编辑CRUSH map之前，最好备份一份原来的CRUSH map
- CRUSH map是可以自定义的，用户应该根据自己的实际情况进行定制
- 规划好CRUSH map，可以让运维工作事半功倍
- 不同rule规则的设定对计算资源的消耗是有差异的
- 同一个CRUSH规则可以应用到多个pool，但是一个pool只能对应一个CRUSH规则
- object 映射到osd的计算过程是在客户端完成的，计算出结果后，客户端直接与对应的osd交互，存取数据，这也是ceph高性能的原因之一