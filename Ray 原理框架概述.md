# Ray 原理框架概述
论文链接：<https://arxiv.org/pdf/1712.05889.pdf>
## 1 Ray 基本架构
Ray的架构由**应用层**和**系统层**组成，其中应用层实现了Ray的API，作为前端供用户使用，而系统层则作为后端来保障Ray的高可扩展性和容错性。
* **应用层：** 实现 API 的应用层，现在包括 Python 和 Java分别实现的版本。
* **系统层：** 提供高扩展性和容错的系统层，用 C++ 写的，以CPython的形式嵌入包中。
 
整体的架构图如下图所示：

<div align=center>
    <img src="https://github.com/Da-jiao-niu/Ray/blob/main/image/Ray%E6%A1%86%E6%9E%B6.png" width="60%">
</div>

### 应用层
应用层中有三种类型的进程：
* **驱动器进程 (Driver Process):** 执行用户程序的进程。顾名思义，所有操作都需要由主进程来驱动。
* **工作器进程 (Worker Process):** 执行由驱动器或其他工作器调用的任务（远程函数）的无状态的进程。工作器由系统层分配任务并自动启动。当声明一个远程函数时，该函数将被自动发送到所有的工作器中。在同一个工作器中，任务是串行地执行的，工作器并不维护其任务与任务之间的局部状态，即在工作器中，一个远程函数执行完后，其局部作用域的所有变量将不再能被其他任务所访问。
* **行动器进程 (Actor Process):** 行动器被调用时只执行其所暴露的方法。行动器由工作器或驱动器显式地进行实例化。与工作器相同的是，行动器也会串行地执行任务，不同的是行动器上执行的每个方法都依赖于其前面所执行的方法所导致的状态。

三种进程体现到Python代码中如下：
```ruby
@ray.remote
def f(x):
    # ==== 工作器进程 ====
    return x * x

@ray.remote
class Counter(object):
    def __init__(self):
        # ==== 行动器进程 ====
        self.value = 0

    def increment(self):
        # ==== 行动器进程 ====
        self.value += 1
        return self.value

if __name__ == "__main__":
    # ==== 驱动器进程 ====
    object_ref = f.remote(2)
    assert ray.get(object_ref) == 4

    counter = Counter.remote()
    refs = []
    for i in range(10):
        ref = counter.increment.remote()
        refs.append(ref)
    for i, ref in enumerate(refs):
        assert ray.get(ref) == i + 1
```

### 系统层
系统层由三个主要部件组成：**全局控制存储器 (Global Control Store)、分布式调度器 (Distributed Scheduler)** 和 **分布式对象存储器 (Distributed Object Store)**。这些部件在横向上是可扩展的，即可以增减这些部件的数量，同时还具有一定的容错性。
#### 全局控制存储(GCS)
GCS设计的初衷是让系统中的各个组件都变得尽可能地无状态，因此GCS维护了一些全局状态：
* **对象表 (Object Table)：** 记录每个对象存在于哪些节点
* **任务表 (Task Table)：** 记录每个任务运行于哪个节点
* **函数表 (Function Table)：** 记录用户进程中定义的远程函数
* **事件日志 (Event Logs)：** 记录任务运行日志

#### 分布式调度器(Distributed Scheduler)
Ray中的任务调度器被分为两层，由一个**全局调度器**和每个节点各自的**局部调度器**组成。为了避免全局调度器负载过重，在节点创建的任务首先被提交到局部调度器，如果该节点没有过载且节点资源能够满足任务的需求（如GPU的需求），则任务将在本地被调度，否则任务才会被传递到全局调度器，考虑将任务调度到远端。由于Ray首先考虑在本地调度，本地不满足要求才考虑在远端调用，因此这样的调度方式也被称为**自底向上的调度**。

下图展示了Ray的调度过程，箭头的粗细表示过程发生频率的高低。用户进程和工作器向本地调度器提交任务，大多数情况下，任务将在本地被调度。少数情况下，局部调度器会向全局调度器提交任务，并向GCS传递任务的相关信息，将任务涉及的对象和函数存入全局的对象表和函数表中，然后全局调度器会从GCS中读取到信息，并选择在其他合适的节点上调度这一任务。更具体地来说，全局调度器会根据任务的请求选出具有足够资源的一系列节点，并在这些节点中选出等待时间最短的一个节点。

<div align=center>
    <img src="https://github.com/Da-jiao-niu/Ray/blob/main/image/%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B.png" width="60%">
</div>

#### 分布式对象存储器 (Distributed Object Store)
Ray实现了一个内存式的分布式存储系统来存储每个任务的输入和输出。Ray通过内存共享机制在每个节点上实现了一个对象存储器 (Object Store)，从而使在同一个节点运行的任务之间不需要拷贝就可以共享数据。当一个任务的输入不在本地时，则会在执行之前将它的输入复制到本地的对象存储器中。同样地，任务总会将输出写入到本地的对象存储器中。这样的复制机制可以减少任务的执行时间，因为任务永远只会从本地对象存储器中读取数据（否则任务不会被调度），并且消除了热数据可能带来的潜在的瓶颈。

## 2 进程视角的架构分析
现在假设有一个求两数之和的任务需要交给Ray来执行，具体分析一下这一任务在Ray的架构中是如何执行的。

<div align=center>
    <img src="https://github.com/Da-jiao-niu/Ray/blob/main/image/%E8%BF%9B%E7%A8%8B%E5%88%86%E6%9E%90a.png" width="60%">
</div>

图(a)描述了任务的定义、提交和执行的过程:

0. **定义远程函数** 位于N1的用户程序中定义的远程函数add被装载到GCS的函数表中,位于N2的工作器从GCS中读取并装载远程函数add。
1. **提交任务** 位于N1的用户程序向本地调度器提交add(a, b)的任务。
2. **提交任务到全局** 本地调度器将任务提交至全局调度器。
3. **检查对象表** 全局调度器从GCS中找到add任务所需的实参a, b，发现a在N1上，b在N2上（a, b 已在用户程序中事先定义）。
4. **执行全局调度** 由上一步可知，任务的输入平均地分布在两个节点，因此全局调度器随机选择一个节点进行调度，此处选择了N2。
5. **检查任务输入** N2的局部调度器检查任务所需的对象是否都在N2的本地对象存储器中。
6. **查询缺失输入** N2的局部调度器发现任务所需的a不在N2中，在GCS中查找后发现a在N1中。
7. **对象复制** 将a从N1复制到N2。
8. **执行局部调度** 在N2的工作器上执行add(a, b)的任务。
9. **访问对象存储器** add(a, b)访问局部对象存储器中相应的对象。

<div align=center>
    <img src="https://github.com/Da-jiao-niu/Ray/blob/main/image/%E8%BF%9B%E7%A8%8B%E5%88%86%E6%9E%90b.png" width="60%">
</div>

图(b)描述了获取任务执行结果的的过程：
1. **提交get请求** 向本地调度器提交ray.get的请求，期望获取add任务执行的返回值。
2. **注册回调函数** N1本地没有存储返回值，所以根据返回值对象的引用id_c在GCS的对象表中查询该对象位于哪个节点，假设此时任务没有执行完成，那么对象表中找不到id_c，因此N1的对象存储器会注册一个回调函数，当GCS对象表中出现id_c时触发该回调，将c从对应的节点复制到N1上。
3. **任务执行完毕** N2上的add任务执行完成，返回值c被存储到N2的对象存储器中。
4. **将对象同步到GCS** N2将c及其引用id_c存入GCS的对象表中。
5. **触发回调函数** [2]中注册的回调函数被触发。
6. **执行回调函数** 将c从N2复制到N1。
7. **返回用户程序** 将c返回给用户程序，任务结束。

## 3 Ray 编程模型
### Ray 的初始化
Ray由一个**头节点（Head node）** 和一组**工作节点（Worker node）** 组成。启动Ray需要首先启动头节点，并为工作节点提供头节点的地址以形成集群。头节点负责管理和分配工作节点的任务，工作节点负责执行任务并返回结果。经过测试，头节点和工作节点可以为同一台计算机。

Ray的启动由两个步骤组成：启动头节点、注册工作节点到头节点。

<div align=center>
    <img src="https://github.com/Da-jiao-niu/Ray/blob/main/image/Ray%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png" width="50%">
</div>

以下是头节点的启动代码和关闭代码：
```ruby
import ray
ray.init()  # 启动
assert ray.is_initialized() == True

ray.shutdown()  # 关闭
assert ray.is_initialized() == False
```
Ray框架采用Actor模型，相较于传统的共享内存模型，Ray不存在状态竞争、可以方便的组建集群、能更好的控制状态。每个Actor即每个工作节点的注册方式如下:
```ruby
import ray
ray.init(address=头节点地址)  # 启动
assert ray.is_initialized() == True

ray.shutdown()  # 关闭
assert ray.is_initialized() == False
```

### Ray 的基本API
Ray中有两个重要的概念：**任务(Task)** 和 **行动器(Actor)**。Ray编程模型是指Ray框架基于任务和行动器这两个重要需求所向用户提供的一套API及其编程范式。以下是Ray的一些基本API：
* ***futures = f.remote(args):*** 远程地执行函数f。f.remote()以普通对象或future对象作为输入，返回一个或多个future对象，非阻塞执行。
* ***objects = ray.get(futures):*** 返回与一个或多个future对象相关联的真实值，阻塞执行。
* ***ready_futures = ray.wait(futures, k, timeout):*** 当futures中有k个future完成时，或执行时间超过timeout时，返回futures中已经执行完的future。

任务是指在无状态的工作器中执行的远程函数。远程函数被调用时会立即返回一个future对象，而真正的返回值可以通过ray.get(<future对象>)的方式来获取。这样的编程模型既允许用户编写并行计算代码，同时又提醒用户要关注数据之间的依赖性。

任务的编程范式如下：
1. 注册任务：在需要注册为任务的函数上加上@ray.remote装饰器
2. 提交任务：在调用具有@ray.remote装饰器的函数时，需要带上.remote()而不是直接调用
3. 非阻塞提交：无论任务的运行需要多少时间，在提交任务后都会立即返回一个ObjectRef对象
4. 按需阻塞获取结果：在你需要函数的返回值时，可以通过ray.get来获取

以下代码是一个任务从注册到运行完成获得结果的示例：
```ruby
@ray.remote
def f(x):
    return x * x

object_ref = f.remote(2)
assert ray.get(object_ref) == 4
```

参考资源：
1. <https://zhuanlan.zhihu.com/p/460600694>
2. <https://zhuanlan.zhihu.com/p/363146146>
3. <https://blog.csdn.net/weixin_43255962/article/details/88689665>
