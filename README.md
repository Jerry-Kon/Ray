# Ray
## Ray编程模型
Ray中有两个重要的概念：任务(Task)和行动器(Actor)。Ray编程模型是指Ray框架基于任务和行动器这两个重要需求所向用户提供的一套API及其编程范式。以下是Ray的一些基本API：
* futures = f.remote(args): 远程地执行函数f。f.remote()以普通对象或future对象作为输入，返回一个或多个future对象，非阻塞执行。
* objects = ray.get(futures): 返回与一个或多个future对象相关联的真实值，阻塞执行。
* ready_futures = ray.wait(futures, k, timeout): 当futures中有k个future完成时，或执行时间超过timeout时，返回futures中已经执行完的future。
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
## Ray的架构
Ray的架构由应用层和系统层组成，其中应用层实现了Ray的API，作为前端供用户使用，而系统层则作为后端来保障Ray的高可扩展性和容错性。整体的架构图如下图所示：

<div align=center>
    
![image](https://github.com/Da-jiao-niu/Ray/blob/main/image/Ray%E6%A1%86%E6%9E%B6.png)
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
系统层由三个主要部件组成：全局控制存储器 (Global Control Store)、分布式调度器 (Distributed Scheduler)和分布式对象存储器 (Distributed Object Store)。这些部件在横向上是可扩展的，即可以增减这些部件的数量，同时还具有一定的容错性。
#### 全局控制存储(GCS)
GCS设计的初衷是让系统中的各个组件都变得尽可能地无状态，因此GCS维护了一些全局状态：
* 对象表 (Object Table)：记录每个对象存在于哪些节点
* 任务表 (Task Table)：记录每个任务运行于哪个节点
* 函数表 (Function Table)：记录用户进程中定义的远程函数
* 事件日志 (Event Logs)：记录任务运行日志

#### 分布式调度器(Distributed Scheduler)
Ray中的任务调度器被分为两层，由一个全局调度器和每个节点各自的局部调度器组成。为了避免全局调度器负载过重，在节点创建的任务首先被提交到局部调度器，如果该节点没有过载且节点资源能够满足任务的需求（如GPU的需求），则任务将在本地被调度，否则任务才会被传递到全局调度器，考虑将任务调度到远端。由于Ray首先考虑在本地调度，本地不满足要求才考虑在远端调用，因此这样的调度方式也被称为自底向上的调度。
下图展示了Ray的调度过程，箭头的粗细表示过程发生频率的高低。用户进程和工作器向本地调度器提交任务，大多数情况下，任务将在本地被调度。少数情况下，局部调度器会向全局调度器提交任务，并向GCS传递任务的相关信息，将任务涉及的对象和函数存入全局的对象表和函数表中，然后全局调度器会从GCS中读取到信息，并选择在其他合适的节点上调度这一任务。更具体地来说，全局调度器会根据任务的请求选出具有足够资源的一系列节点，并在这些节点中选出等待时间最短的一个节点。

<div align=center>
    
![image](https://github.com/Da-jiao-niu/Ray/blob/main/image/Ray%E6%A1%86%E6%9E%B6.png)
</div>

#### 分布式对象存储器 (Distributed Object Store)
