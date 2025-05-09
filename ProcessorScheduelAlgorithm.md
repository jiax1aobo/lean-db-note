- 谈一谈对进程调度算法的了解

# 进程调度算法

## 先来先服务（First Come First Served）

**非抢占式**，所有进程按照先后顺序排队等待执行。如果短作业排在了长作业后面，将等待过久。

## 短作业优先（Shortest Job First）

**非抢占式**，所有进程按照运行时间最短的顺序排队。如果不停有短作业到来，将导致长作业一直等待（饿死）。

## 最短剩余时间优先（Shortest Remaining Time First）

**SJF的抢占式版本**，按进程的剩余运行时间调度。

## 时间片轮转

**非抢占式**，所有进程按先后顺序排队，将CPU时间片分给队首作业，时间片用完就把当前队首作业排到队尾。

## 优先级调度

为每个进程划分优先级，根据优先级调度。低优先级的作业随等待时间增加而提升优先级。

## 多级反馈队列

**时间片轮转和优先级调度的结合**，设置多个队列，每个队列的时间片大小不同，如指数级增加。时间片小的队列优先级更高，进程首先在时间片小的队列排队执行，如果时间片结束后未执行完就被移动到下一级时间片的队首。因为队列是按时间片长短划分优先级的，因此上一个队列没有进程在排队才能调度当前队列中的进程。