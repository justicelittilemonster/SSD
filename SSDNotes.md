# Chapter 1
### 闪存不能覆盖写，闪存块需擦除才能写入
### FTL完成逻辑数据块到闪存物理空间的转换或者映射  以及其他的事情 例如垃圾回收。

### 磨损均衡： 每个闪存块不能一直写数据。FTL必须尽量让每个闪存块均衡写入


### 对SSD来说 满盘后 写会触发垃圾回收，导致写性能下降 

> ## BIOS和SSD的连接过程
> ### 1.和SSD发生链接，协商到正确的速度上
> ### 2.发出识别盘的命令。
> ### 3.读取基本信息，直到BIOS找到主引导记录MBR
> ### 4.MBR开始读取硬盘分区表DPT，找到分区引导记录PBR，并把控制权交给PBR
> ### 5.SSD通过数据读写功能完成最后的OS加载

---
---

# Chapter 2 SSD主控和全闪存阵列
### 后端两大模块分别为EEC模块和闪存控制器 
### EEC：数据编解码单元 闪存存储天生存在误码率，在写数据时应给原数据加入EEC校验保护，这是编码。读数据时需要通过解码来检错和纠错。 

---
---
# Chapter 3 闪存物理结构
### DIE是接收和执行闪存命令的基本单元。DIE1和DIE2可以同时执行和接收不同的指令。但在一个DIE中，一次只能同行 执行一个指令

### 无论是从闪存介质读数据到Cache Register，还是把Cache register的数据写入闪存介质，都已Page为单位
### 闪存写入时间是指一个page的数据从page register当中写入闪存介质的时间。读相反。
### 闪存的擦除以Block为单位，因为一个Block当中的所有存储单元是共用一个衬底的
### Charge Trap型闪存：将浮栅晶体管的浮栅层换成绝缘层来存储电荷
### 对于MLC与TLC而言，写一个闪存块当中的闪存页，应该顺序写，禁止随机写。因为：
> ### 1.一个存储单元包含两个闪存页数据，要先写lower page，再写Upper Page
> ### 2.相邻单元之间有耦合电容，工艺上要求后面的闪存页写操作时前面的闪存页已经写过。
### *但对读没有这个限制*

### 对MLC来说，一个存储单元对应着两个page 假设lower page先写，然后在写Upper Page的过程中，如果这个时候掉电，那么值钱写入的lower page也会丢失。也就是说，写一个闪存页失败，可能会导致另外一个闪存页的数据损坏

## 读干扰
### 读操作时，衬底加电压是0，读取的page，控制极上的电压是Vref，而对其他Page，控制极上加的电压是Vpass。问题来了，由于其他Page上加了一个Vpass，相对来说是一个比较大的电压，这样就会在浮栅极和衬底形成一个较强的电场，可能就把一些电子吸入浮栅极，有点轻微Program的意思。随着该Block上Page数据读的次数越来越多，无辜的page（没有被读到的）中的浮栅极电子进入越来越多，导致里面数据状态发生变化，也就意味着数据出错。

## 读错误来源：
> ### 1.擦写次数增多：氧化层逐渐老化，电子进出存储单元越来越容易。电荷容易发生异常。
> ### 2.Data Retention:存储在存储单元的电子会流失，整个阈值电压分布向左移动，导致读数据的时候发生误判
> ### 3.读干扰
> ### 4.存储单元之间干扰：存储电子的浮栅极是导体，两个导体之间构成电容，一个存储单元电荷的变化会导致其他存储单元电荷变化
> ### 5.写错误。如果写Upper Page基于Lower Page的状态。前一个出错会导致Upper Page出错


### ECC纠错码
### RAID 类似磁盘阵列 Die p存储校验数据，为Die 0、1、2、3、提供异或，能恢复其中一个Die上的数据。 采用冗余纠错技术，需要额外的空间

### 数据随机化 让0和1的分布均匀，不断地输入全0或全1，很容易导致闪存内部电量不均衡
---
---
# Chapter 4 FTL

### Flash Translation Layer（闪存转换层） 完成主机逻辑地址空间到闪存物理地址空间的映射

###  MLC或TLC的读写速度都不如SLC，但他们都可以配成SLC模式来使用

### FTL分为Host Based和Device Based 
> ### Host Based （基于主机）用的是自己计算机的CPU和内存资源。
> ### Device Based（基于设备） 用的是SSD上的控制器和RAM资源


## 映射管理 
### 块映射：以闪存的块为映射粒度。用户即使只写入一个逻辑页，也要把整个物理块数据先读出来。改变后再写入。有好的大尺寸的读写性能，但小尺寸数据很慢。
### 页映射：
### 混合映射


### HMB  映射表除了可以放在板载DRAM，SRAM和闪存中，它还可以放在主机的内存中。
### 映射表在SSD掉电前，需要把它写到闪存中。
### 写放大：  由于GC的存在，用户要写数据，SSD需要做一些额外的数据搬移。导致SSD往闪存中写入的数据量比实际用户写入的多
### 垃圾回收步骤：
> ### 1.挑选源闪存块
> ### 2.从源闪存块中找有效数据
> ### 3.把有效数据写入到目标闪存块
###  Trim 当用户删除文件时，其实只是切断用户与操作系统的联系，而在SSD内部，逻辑页与物理页的映射关系还在。在没有Trim之前，SSD无法知道哪些数据是无效的。在做垃圾回收的时候。仍把他当作有效数据进行数据的搬移。 Trim让SSD知道数据无效，可做垃圾回收。
### 磨损均衡：让SSD中每个闪存块的磨损都保持均衡
> ### 动态磨损平衡：把热数据写到年轻的块上
> ### 静态磨损平衡：把冷数据搬到擦写次数比较多的闪存块上。

### 掉电恢复：
## 正常掉电：
> ### 把buffer中缓存的用户数据刷入闪存
> ### 把映射表刷入闪存
> ### 把闪存的块信息写入闪存（写过哪些闪存块等信息）
> ### 把SSD其他信息写入闪存
## 异常掉电：
### SSD的异常掉电恢复就是映射表的恢复重建

### 坏块管理策略
> ### 1.略过策略 
> ### 2.替换策略：当某个Die上发现块块时，会被预留空间中的某个好块替换

### SLC cache 把MLC或TLC里面的一些闪存块配置成SLC模式来访问 一般用在消费级SSD


# Chapter 5 PCIe介绍
### PICe采用的是树形拓扑结构。
### PCIe定义了下三层：事务层、数据链路层和物理层
### 事务层的主要职责是创建或解析TLP(Transaction Layer Packet)、流量控制，QoS等
###  数据链路层的主要职责是创建或解析DLLP（Data Link Layer Packet），Ack/Nak协议
