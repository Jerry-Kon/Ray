# Ray
## Ray编程模型
Ray中有两个重要的概念：任务(Task)和行动器(Actor)。Ray编程模型是指Ray框架基于任务和行动器这两个重要需求所向用户提供的一套API及其编程范式。以下是Ray的一些基本API：
* futures = f.remote(args): 远程地执行函数f。f.remote()以普通对象或future对象作为输入，返回一个或多个future对象，非阻塞执行。
* objects = ray.get(futures): 返回与一个或多个future对象相关联的真实值，阻塞执行；
* ready_futures = ray.wait(futures, k, timeout): 当futures中有k个future完成时，或执行时间超过timeout时，返回futures中已经执行完的future
