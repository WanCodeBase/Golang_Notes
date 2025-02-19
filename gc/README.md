# GC 垃圾回收

=================

* [GC 垃圾回收](#gc-垃圾回收)
    * [垃圾回收算法](#垃圾回收算法)
        * [标记\-清除法（Go 1\.3）](#标记-清除法go-13)
        * [三色标记法（Go 1\.5）](#三色标记法go-15)
        * [三色标记法\+写屏障（Go 1\.8）](#三色标记法写屏障go-18)
    * [GC的触发条件](#gc的触发条件)
    * [GC调优](#gc调优)
* [返回](../README.md)

## 垃圾回收算法
### 标记-清除法（Go 1.3）
1. 暂停业务逻辑，找到不可大对象和可达对象
2. 开始标记，程序找出所有可达对象并标记
3. 标记完成后，清除未标记对象
4. 停止暂停，让程序继续跑。
5. 循环以上步骤，直到程序生命周期结束
  
**缺点**
- STW，让程序暂停，程序出现卡顿
- 标记需要扫描整个堆
- 清除数据会产生堆碎片
### 三色标记法（Go 1.5）
1. 新创建的对象，默认为白色
2. GC回收开始，从根节点遍历对象，把白色标记为灰色
3. 遍历灰色集合，将灰色集合引用的白色对象放入灰色集合，再将该灰色对象放入黑色集合
4. 重复第三步，直到灰色集合中无对象
5. 清除白色标记的对象  

**缺点**
- 为了避免GC过程中，对象之间的引用关系产生新的变更，使得GC的结果发生错误，需要STW（stop the world)

**引用对象丢失的原因**  
一个黑色的节点A新增了指向白色节点C的引用，并且白色节点C除了A之外没有其他灰色节点的引用，或者存在但是在GC过程中被删除了。以上两个条件同时满足时，就会出现对象丢失。

**解决方案**  
1. 插入屏障（强三色不变式）：强制不允许黑色对象引用白色对象
2. 删除屏障（弱三色不变式）：被删除的对象，如果自身为灰色或者白色，那么标记为灰色
### 三色标记法+写屏障（Go 1.8）
1. GC开始时将栈上的可达对象全部标记为黑色
2. GC期间，栈上创建的新对象，都标记为黑色
3. 堆上被删除的对象标记为灰色
4. 堆上新创建的对象标记为灰色

**缺点**
- STW,引入写屏障保护可以减少暂停时间
- 需要额外为每个对象维护状态信息，增加内存开销
- 需要频繁的迭代标记和清除对象，如果回收的垃圾对象很对，会增加耗时
- 回收过程中，如果频繁的进行对象的移动和重新分配内存，会导致内存碎片化，降低内存的利用率

## GC的触发条件
- 主动触发（手动触发）：runtime.GC 此调用阻塞式等待当前GC运行完毕
- 被动触发：
  - 步调算法（Pacing）控制内存增长比率，每次内存分配时检查当前内存分配量是否已达到阈值，达到阈值则启动GC
  - 使用系统监控，当超过两分钟没有产生任何GC时，就强制触发GC

## GC调优
- 控制内存分配的速度，限制goroutine的数量
- 少量使用+链接string
- 切片提前分配足够的内存来降低扩容带来的拷贝
- 避免map key对象过多，导致扫描时间增加
- 变量复用，减少对象分配
- 增大GOGC的值，降低GC运行的频率