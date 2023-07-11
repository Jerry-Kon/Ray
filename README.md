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

'''ruby
hello word
'''
