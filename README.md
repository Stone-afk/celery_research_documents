# Celery 分布式任务框架调研

### 简介

Celery 通过消息机制进行通信，通常使用中间人（Broker）作为客户端和职程（Worker）调节。启动一个任务，客户端向消息队列发送一条消息，然后中间人（Broker）将消息传递给一个职程（Worker），最后由职程（Worker）进行执行中间人（Broker）分配的任务。

Celery 可以有多个职程（Worker）和中间人（Broker），用来提高Celery的高可用性以及横向扩展能力。

Celery 是用 Python 编写的，但协议可以用任何语言实现。除了 Python 语言实现之外，还有Node.js的[node-celery](https://github.com/mher/node-celery)和php的[celery-php](https://github.com/gjedeer/celery-php)。

可以通过暴露 HTTP 的方式进行，任务交互以及其它语言的集成开发。

**我们需要什么？**

Celery 需要消息中间件来进行发送和接收消息。 RabbitMQ 和 Redis 中间人的功能比较齐全，但也支持其它的实验性的解决方案，其中包括 SQLite 进行本地开发。

Celery 可以在一台机器上运行，也可以在多台机器上运行，甚至可以跨数据中心运行。



### 架构

Celery的架构由三部分组成，消息中间件（message broker）、任务执行单元（worker）和 任务执行结果存储（backend - task result store）组成。

- #### 消息中间件

  Celery本身不提供消息服务，但是可以方便的和第三方提供的消息中间件集成。包括，RabbitMQ, Redis等等

- #### 任务执行单元

  Worker是Celery提供的任务执行的单元，worker并发的运行在分布式的系统节点中。

- #### 任务结果存储

  Task result store用来存储Worker执行的任务的结果，Celery支持以不同方式存储任务的结果，包括AMQP, redis等

![image-20221230001620814](docs/image-20221230001620814.png)

![image-20221230001222151](docs/image-20221230001222151.png)



### 基本原理

![image-20221230001457919](docs/image-20221230001457919.png)



- 准备配置了broker与backend的worker(任务的来源)，并启动。
- 添加任务到broker，worker就会执行任务，将结果存储到backend中。
- 想查看任务的执行结果，根据任务的id去bckend中查询。



### 工作流

```
+-----------+
                          | Signature |
                          +------^----+
                                 |
      +------------+-------------+-------------+----------+--------------+
      |            |             |             |          |              |
      |            |             |             |          |              |
+-----+-+     +----+---+    +-------+     +----+--+    +--+---+    +-----+----+
| chain |     | chunks |    | group |     | chord |    | xmap |    | xstarmap |
+-------+     +--------+    +-------+     +-------+    +------+    +----------+
```

##### Signature

使用 Celery Signature 签名（Subtask 子任务），可生成一个特殊的对象——任务签名。任务签名和函数签名类似，除了包含函数的常规声明信息（形参、返回值）之外，还包含了执行该函数所需要的调用约定和（全部/部分）实参列表。你可以在任意位置直接使用该签名，甚至不需要考虑实参传递的问题（实参可能在生成签名时就已经包含）。可见，任务签名的这种特性能够让不同任务间的组合、嵌套、调配变得简单。

**任务签名支持「直接执行」和「Worker 执行」两种方式：**

- **生成任务签名并直接执行**：签名在当前进程中执行

  ```python
  >>> from celery import signature
  >>> from proj.task.tasks import add
  
  # 方式一
  >>> signature('proj.task.tasks.add', args=(2, 3), countdown=10)
  proj.task.tasks.add(2, 3)
  # 这个签名包括了任务的名称是 proj.task.tasks.add， 2个参数，超时时间设置为10s
  >>> s_add = signature('proj.task.tasks.add', args=(2, 3), countdown=10)
  >>> s_add()
  5
  
  # 方式二
  >>> add.signature((3, 4), countdown=10)
  proj.task.tasks.add(3, 4)
  >>> s_add = add.signature((3, 4), countdown=10)
  >>> s_add()
  7
  
  # 方式三
  >>> add.subtask((3, 4), countdown=10)
  proj.task.tasks.add(3, 4)
  >>> s_add = add.subtask((3, 4), countdown=10)
  >>> s_add()
  7
  
  # 方式四
  >>> add.s(3, 4)
  proj.task.tasks.add(3, 4)
  >>> s_add = add.s(3, 4)
  >>> s_add()
  7
  ```

- **生成任务签名并交由 Worker 执行**：签名在 Worker 服务进程中执行

```python
# 调用 delay/apply_async 方法将签名加载到 Worker 中执行
>>> s_add = add.s(2, 2)
>>> s_add.delay()       
<AsyncResult: 75be3776-b36b-458e-9a89-512121cdaa32>
>>> s_add.apply_async()
<AsyncResult: 4f1bf824-331a-42c0-9580-48b5a45c2f7a>


>>> s_add = add.s(2, 2)
>>> s_add.delay(debug=True)     # 任务签名支持动态传递实参
<AsyncResult: 1a1c97c5-8e81-4871-bb8d-def39eb539fc>
>>> s_add.apply_async(kwargs={'debug': True})
<AsyncResult: 36d10f10-3e6f-46c4-9dde-d2eabb24c61c>
```

##### Primitive

- group 可以并行处理的一组任务

    - 任务组函数接收一组任务签名列表，返回一个新的任务签名——签名组，调用签名组会并行执行其包含的所有任务签名，并返回所有结果的列表。常用于一次性创建多个任务

      ```python
      >>> from celery import group
      >>> from proj.task.tasks import add
      >>> add_group_sig = group(add.s(i, i) for i in range(10))
      >>> result = add_group_sig.delay()
      >>> result.get()
      [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
      # 返回多个结果
      >>> result.results
      [<AsyncResult: 1716cfd0-e87c-4b3d-a79f-1112958111b1>, 
       <AsyncResult: a7a18bde-726e-49b2-88ed-aeba5d3bf5f2>, 
       <AsyncResult: b9d9c538-2fad-475a-b3d1-bd1488278ce2>, 
       <AsyncResult: 6f370fdd-ed7e-430a-a335-af4650ca15cf>, 
       <AsyncResult: a6ddbe14-5fbd-4079-9f12-35ebbc89d89b>, 
       <AsyncResult: 65dece11-9f38-4940-9fa0-7fcf09266c7a>, 
       <AsyncResult: 8205ffc0-1056-469a-a642-96676d1518e7>, 
       <AsyncResult: e77b7e2b-66d2-48b8-9ffd-4f8fa7d9f4a4>, 
       <AsyncResult: 355b7d01-72c1-4b00-8572-407e751d76c3>, 
       <AsyncResult: aa561ac3-656f-4c81-9e3c-00c64ca49181>] 
      ```

- chain 将任务组装成一个调用链

    - 任务链函数接收若干个任务签名，并返回一个新的任务签名——链签名。调用链签名会并串行执行其所含有的任务签名，每个任务签名的执行结果都会作为第一个实参传递给下一个任务签名，最后只返回一个结果。

      ```python
      >>> from celery import chain
      >>> from proj.task.tasks import add
      >>> add_chain_sig = chain(add.s(1, 2), add.s(3))
      # 精简语法
      >>> add_chain_sig = (add.s(1, 2) | add.s(3))
      >>> result = add_chain_sig.delay()          # ((1 + 2) + 3)
      >>> result.status
      u’SUCCESS’
      >>> result.get()
      6
      # 仅返回最终结果
      >>> result.results
      Traceback (most recent call last):
        File "<stdin>", line 1, in <module>
      AttributeError: 'AsyncResult' object has no attribute 'results'
      # 结合偏函数
      >>> add_chain_sig = chain(add.s(1), add.s(3))
      >>> result = add_chain_sig.delay(3)        # ((3 + 1) + 3)
      >>> result.get()
      7
      ```

- chord 可以使用Map-Reduce的方式执行任务

    - 复合任务函数生成一个任务签名时，会先执行一个组签名（不支持链签名），等待任务组全部完成时执行一个回调函数。

      ```python
      >>> from proj.task.tasks import add, log
      >>> from celery import chord, group, chain
      >>> add_chord_sig = chord(group(add.s(i, i) for i in range(10)), log.s())
      >>> result = add_chord_sig.delay()
      >>> result.status
      u'SUCCESS'
      >>> result.get()
      u'LOG: [0, 2, 4, 6, 8, 10, 12, 16, 14, 18]'
      ```

- map 使用map方式执行系列任务

    - 映射函数，与 Python 函数式编程中的 map 内置函数相似。都是将序列对象中的元素作为实参依次传递给一个特定的函数。如果使用 map 来处理 add 函数会报错，因为 map 只能支持一个参数的传入。

      ```python
      >>> add.map(zip(range(10), range(100)))
      [proj.task.tasks.add(x) for x in [(0, 0), (1, 1), (2, 2), (3, 3), (4, 4), (5, 5), (6, 6), (7, 7), (8, 8), (9, 9)]]
      >>> result = add.map(zip(range(10), range(100))).delay(1)
      >>> result.status
      u’FAILURE'
      ```

- starmap 类似map方式，支持*args

    - map 和 starmap 的区别在于，前者的参数只有一个，后者支持的参数有多个。

      ```python
      >>> add.starmap(zip(range(10), range(100)))
      [proj.task.tasks.add(*x) for x in [(0, 0), (1, 1), (2, 2), (3, 3), (4, 4), (5, 5), (6, 6), (7, 7), (8, 8), (9, 9)]]
      >>> result = add.starmap(zip(range(10), range(100))).delay()
      >>> result.get()
      [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
      ```

- chunks 将任务拆分成块进行执行

    - 任务块函数能够让你将需要处理的大量对象分为分成若干个任务块，如果你有一百万个对象，那么你可以创建 10 个任务块，每个任务块处理十万个对象。有些人可能会担心，分块处理会导致并行性能下降，实际上，由于避免了消息传递的开销，因此反而会大大的提高性能。

      ```python
      >>> add_chunks_sig = add.chunks(zip(range(100), range(100)), 10)
      >>> result = add_chunks_sig.delay()
      >>> result.get()
      [[0, 2, 4, 6, 8, 10, 12, 14, 16, 18], 
       [20, 22, 24, 26, 28, 30, 32, 34, 36, 38], 
       [40, 42, 44, 46, 48, 50, 52, 54, 56, 58], 
       [60, 62, 64, 66, 68, 70, 72, 74, 76, 78], 
       [80, 82, 84, 86, 88, 90, 92, 94, 96, 98], 
       [100, 102, 104, 106, 108, 110, 112, 114, 116, 118], 
       [120, 122, 124, 126, 128, 130, 132, 134, 136, 138], 
       [140, 142, 144, 146, 148, 150, 152, 154, 156, 158], 
       [160, 162, 164, 166, 168, 170, 172, 174, 176, 178], 
       [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]
      ```



### 任务形态

- 本地任务



### 任务分类

celery 一共支持以下三种任务，只支持手动触发或配置

- 异步定时任务

- 定时任务

- 一次性异步任务



### 任务编排

- 暂未支持



### 任务注册

- 通过代码的形式注册，最简单最常见的 **一次性异步任务** 代码注册如下：

  ```python
  import time
  from celery import Celery
  
  #消息中间件（使用的redis）
  broker = 'redis://localhost:6379/1'
  #结果存储（使用的redis）
  backend = 'redis://localhost:6379/2'
  #实例化Celery对象
  app = Celery(
      'celeryDemo',
      broker=broker,
      backend=backend
  )
  
  # 添加@app.task()装饰器，说明执行的任务是一个异步任务
  @app.task()
  def add(x,y):
      print('task enter ....')
      time.sleep(5)
      return x+y
  ```

  在使用该任务的文件中

  ```python
  from tasks import add
  
  if __name__ == '__main__':
      print('task start....')
      result = add.delay(2,3)
      print('task end....')
      print(result)
  ```



### 任务管理

#### 任务运行结果

- celery 启动后直接到运行的后台的celery日志查看

- 命令查看

  ```shell
  $ celery -A proj result -t tasks.add 4e196aa4-0141-4601-8138-7aa33db0f577  # uuid 是task id
  ```

- 代码查看

  ```python
  # 返回task_id(由.delay()提供)，然后向celery实例询问状态：
  x = method.delay(1,2)
  print(x.task_id)
  
  # 询问时，使用以下task_id获取新的AsyncResult：
  
  from celery.result import AsyncResult
  res = AsyncResult(x.task_id)
  res.ready()
  ```



#### 回调机制

- on_failure
    - 异步任务执行失败时，会执行这个回调方法。如果你想实现一个 异步任务执行失败时，需要发出告警邮件通知的功能，重写这个函数就可实现。
- on_success
    - 异步任务执行成功时，会执行这个回调方法。
- on_retry
    - 异步任务尝试重试时，会执行这个回调方法。
- after_return
    - 异步任务执行成功，并且return了一些内容，会执行这个回调方法。
- update_state
    - 你可以手动调用这个方法来更新任务状态。
- send_error_email
    - 异步任务执行失败时，并且配置了send_error_emails=True时，会执行这个回调方法。（需要在django中配置相关邮件参数）最新版本已经删掉了这个方法.

```python
from celery import Task
 
class CustomTask(Task):
    def on_success(self, retval, task_id, args, kwargs):
        print('异步任务成功')
        return super(CustomTask, self).on_success(retval, task_id, args, kwargs)
 
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print('异步任务失败', exc)
        return super(CustomTask, self).on_failure(exc, task_id, args, kwargs, einfo)
 
    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        print(retval)
        return super(CustomTask, self).after_return(status, retval, task_id, args, kwargs, einfo)
 
 
@celery_app.task(base=CustomTask)
def send_email():
    print('开始执行发送邮件异步任务')
    return {'message': '成功'}
 
打印结果：
"""
[2021-01-13 15:11:11,266: WARNING/MainProcess] ???[???]: celery@55Y72D0VY2BLTW2 ready.
[2021-01-13 15:11:11,267: DEBUG/MainProcess] basic.qos: prefetch_count->16
[2021-01-13 15:11:13,273: INFO/MainProcess] ???[???]: Received task: send_email.tasks.send_email[91ea495d-2104-41f2-b2ae-800710f451ad]
[2021-01-13 15:11:13,276: WARNING/Worker-1] send_email.tasks.send_email[91ea495d-2104-41f2-b2ae-800710f451ad]: 开始执行发送邮件异步任务
[2021-01-13 15:11:13,279: WARNING/Worker-1] send_email.tasks.send_email[91ea495d-2104-41f2-b2ae-800710f451ad]: 异步任务成功
[2021-01-13 15:11:13,279: WARNING/Worker-1] send_email.tasks.send_email[91ea495d-2104-41f2-b2ae-800710f451ad]: {'message': '成功'}
[2021-01-13 15:11:13,280: INFO/MainProcess] ???[???]: Task send_email.tasks.send_email[91ea495d-2104-41f2-b2ae-800710f451ad] succeeded in 0s: {'message': '成功'}
"""
 
```



#### 任务日志

Celery 中有一个 `celery.task`，可以通过继承的方式获取日志的一部分任务名称以及唯一ID。

最好的做法是在模块顶部为所有任务创建一个共有的日志记录器：

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@app.task
def add(x, y):
  logger.info('Adding {0} + {1}'.format(x, y))
  return x + y

```

Celery 使用的是 Python 标准的日志库，详情文档可以在这里找到。 也可以使用 print() ,所有标准 out/-err 都会被重定向到写入到日志系统中（可以关闭该功能，详情参阅 worker_redirect_stdouts）。

但 celery logging 比较复杂且不易设置。底层的 python 日志记录系统需要支持 celery 支持的所有并发的设置：eventlet，greenlet，threads 等。但现实的情况是现在的 python 日志记录系统并不支持所有这些不同的配置。

celery 在 celery.app.log 中提供的特殊的get_task_logger 功能。这将返回一个继承自记录器celery的特殊记录器 celery.task，该记录器自动获取任务名称以及唯一 ID 作为日志的一部分。

但是，也可以使用标准getlogger方式获取日志记录对象，原因是我们很可能在 celery 或者 web 应用程序中调用代码。如果我们使用 logging.getlogger(**name**)，可以使我们的底层代码与执行代码的上下文保持干净整洁。



#### 删除任务

- #### 取消操作

Celery（本文基于5.1.2版本）有两种取消操作。

一是通过[Control.revoke](https://docs.celeryproject.org/en/stable/reference/celery.app.control.html?highlight=control.revoke#celery.app.control.Control.revoke)：

```python
from celery.app.control.Control import revoke

ctrl = Control()
ctrl.revoke(task_id)
# Or terminate
ctrl.revoke(task_id, terminate=True)
ctrl.terminate(task_id)
```

当然，使用`Control`最简单的办法是直接利用Celery的`app`对象：

```python
app.control.revoke(task_id)
```

二是通过[AsyncResult.revoke](https://docs.celeryproject.org/en/stable/reference/celery.result.html?highlight=revoke#celery.result.AsyncResult.revoke)：

```python
from celery.result import AsyncResult

result = AsyncResult(task_id)
result.revoke()
# Or terminate
result.revoke(terminate=True)
```

二者基本是等价的，只是后者常用于复杂的操作场景，前者只是单纯地取消。 更多关于`revoke`的内容，可以参考[Revoking tasks](https://docs.celeryproject.org/en/latest/userguide/workers.html#revoke-revoking-tasks)。

以下基于[AsyncResult.revoke](https://docs.celeryproject.org/en/stable/reference/celery.result.html?highlight=revoke#celery.result.AsyncResult.revoke)进行介绍，省略写`task_id`。

- #### 取消任务

并非所有任务状态，都可以取消。对于已经结束的任务，取消是无效操作。

![image-20221224195136753](docs/image-20221224195136753.png)

可以被取消的只有：

- PENDING
- STARTED
- RETRY

#### 取消PENDING

这代表Task还在排队中，没有真正运行。只需要使用普通的取消即可。

```python
result.revoke()
```

它的原理是，发广播通知所有worker，更新revoked列表。 当worker执行到revoked列表中的任务时，会直接跳过。

如果一个任务已经在运行，则以上操作无效。 因此，它只适用于取消未运行任务的场景，业务能容忍、或要求已运行任务自然地执行完毕。

#### 取消STARTED

如果一个任务已经在运行，通常处于STARTED状态。 取消这个任务，意味着中断这个运行过程。

```python
result.revoke(terminate=True)
# Or with signal
import signal
result.revoke(terminate=True, signal=signal.SIGTERM)
```

要注意的是，`terminate=True`意味着使用给定的signal，结束任务进程。 这里的signal，可以是[signal](https://docs.python.org/zh-cn/3/library/signal.html)模块的任意值。 默认的`signal=None`，即等价于`signal=signal.SIGTERM`。

在结束任务进程时，任务自己是不知道的。 因此，这只适用于粗暴终止的场景。

然而，`SIGTERM`有时候并没有粗暴到底，会等一会儿，往往不是预期行为。 因此，如果要以终止的方式取消正在运行的任务，也可以使用`SIGQUIT`。

#### 利用SIGUSR1

尽管所有的signal都是可用的，但是有四个是Celery预留的特殊signal，详见[Process Signals](https://docs.celeryproject.org/en/stable/userguide/workers.html#process-signals)。

| Signal    | 原文                                       | 备注                                                         |
| --------- | ------------------------------------------ | ------------------------------------------------------------ |
| `SIGTERM` | Warm shutdown, wait for tasks to complete. | 热停止，默认行为。                                           |
| `SIGQUIT` | Cold shutdown, terminate ASAP              | 冷停止。                                                     |
| `SIGUSR1` | Dump traceback for all active threads.     | 发一个[SoftTimeLimitExceeded](https://docs.celeryproject.org/en/stable/reference/celery.exceptions.html?highlight=Terminated#celery.exceptions.SoftTimeLimitExceeded)异常。 |
| `SIGUSR2` | Remote debug, see celery.contrib.rdb.      | 调试专用。                                                   |

其中，`SIGTERM`就是默认的Signal，很粗暴；`SIGQUIT`则比它更粗暴。 至于不在上面的`SIGKILL`、`SIGINT`等，也不见得好多少。 如果想要让被终止的任务知道自己被终止，目前只能使用`SIGUSR1`。

它会在任务正在执行的位置，抛出一个[SoftTimeLimitExceeded](https://docs.celeryproject.org/en/stable/reference/celery.exceptions.html?highlight=Terminated#celery.exceptions.SoftTimeLimitExceeded)。 因此，只需要在最外层用`try ... except`捕获这个异常，就可以进行一些后处理。 但是，由于它的现象和任务超时是类似的，因此需要自行区分。

#### 结论

对于只取消没有在运行的任务，只需要`revoke`即可； 对于已经在运行的任务，则需要`terminate=True`。 默认的`signal=signal.SIGTERM`，有时结束得不够快，有时过于粗暴，缺少一些后处理。 因此，对不关心其运行情况的业务，也可考虑使用`SIGQUIT`，甚至`SIGKILL`。 对于需要做取消前做一些处理的，可以用`SIGUSR1`。

#### 修改任务

- 在运行的过程种不允许修改调度配置



### 调度模型

celery 的核心架构，分成了**调度器（master/main process）** 和 **工作进程（slaves/worker processes）**，也就是我们常说的主从。

celery 的消费模型很简单，调度器负责任务的获取，分发，工作进程（slaves/worker processes）的管理（创建，增加，关闭，重启，丢弃等等），其他辅助模块的维护等等。工作进程负责消费从调度器传递过来的任务。

**具体流程**：

- **调度器**首先**预生成**（prefork）工作进程, 做为一个**进程池**（mutiprocessing-pool），
- 之后通过**事件驱动**（select/poll/epoll）的方式，监听内核的事件（读、写、异常等等），如果监听到就执行对应的回调，源源不断的从 **中间人**（broker）那里提取任务，
- 并通过 **管道**（pipe）作为进程间通讯的方式，运用一系列的路由策略（round-robin、weight 等等）交给工作进程。
- **工作进程**消费（ack）任务，再通过管道向调度器进行状态同步（sync），进程间通讯等等行为。

**celery 的并发模型主要包括下面一些实现，比如基于fork的多进程，基于eventlet和gevent的协程和多线程等; 而默认使用的是prefork也就是多线程模式**。

并发模型主要包括下面一些实现，比如基于fork的多进程，基于eventlet和gevent的协程和多线程等:

```python
`ALIASES = {
  'prefork': 'celery.concurrency.prefork:TaskPool',    
  'eventlet': 'celery.concurrency.eventlet:TaskPool',    
  'gevent': 'celery.concurrency.gevent:TaskPool',    
  'solo': 'celery.concurrency.solo:TaskPool',    
  'processes': 'celery.concurrency.prefork:TaskPool',  # XXX compat alias    
  'threads': 'celery.concurrency.thread:TaskPool' 
} 
  
  def def get_implementation(cls):
    """Return pool implementation by name."""
    return symbol_by_name(cls, ALIASES)
```

- #### **Consumer-Blueprint实现任务调度**

查看celery的任务处理日志:

```python
[2021-11-24 21:33:50,535: INFO/MainProcess] Received task: myapp.add[e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2]
[2021-11-24 21:33:50,535: DEBUG/MainProcess] TaskPool: Apply <function _trace_task_ret at 0x7fe6086ac280> (args:('myapp.add', 'e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2', {'lang': 'py', 'task': 'myapp.add', 'id': 'e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2', 'shadow': None, 'eta': None, 'expires': None, 'group': None, 'group_index': None, 'retries': 0, 'timelimit': [None, None], 'root_id': 'e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2', 'parent_id': None, 'argsrepr': '(16, 16)', 'kwargsrepr': '{}', 'origin': 'gen83110@192.168.5.28', 'reply_to': '63862dbb-9d82-3bdd-b7fb-03580941362a', 'correlation_id': 'e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2', 'hostname': 'celery@192.168.5.28', 'delivery_info': {'exchange': '', 'routing_key': 'celery', 'priority': 0, 'redelivered': None}, 'args': [16, 16], 'kwargs': {}}, b'[[16, 16], {}, {"callbacks": null, "errbacks": null, "chain": null, "chord": null}]', 'application/json', 'utf-8') kwargs:{})
[2021-11-24 21:33:50,536: DEBUG/MainProcess] Task accepted: myapp.add[e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2] pid:83086
[2021-11-24 21:33:50,537: INFO/ForkPoolWorker-8] Task myapp.add[e9bb4aa0-8280-443f-a5ed-3deb0a0b99c2] succeeded in 0.000271957000000711s: 32

```

可以发现**celery的worker在主进程(MainProcess)中接收到task后，会派发给子进程(ForkPoolWorker-8)执行**。

synloop的消费函数on_message实际上是Consumer(Blueprint)的create_task_handler:

```python
def create_task_handler(self, promise=promise):
    strategies = self.strategies
    on_unknown_message = self.on_unknown_message
    on_unknown_task = self.on_unknown_task
    on_invalid_task = self.on_invalid_task
    callbacks = self.on_task_message
    call_soon = self.call_soon
    
    def on_task_received(message):
        type_ = message.headers['task']  
        ...
        strategy = strategies[type_]
        strategy(
                        message, payload,
                        promise(call_soon, (message.ack_log_error,)),
                        promise(call_soon, (message.reject_log_error,)),
                        callbacks,
                    )
        ...
    return on_task_received
```

对于消息和任务的处理，celery提供了**默认的执行策略 default**:

```python
# celery/worker/strategy.py:22
def default(task, app, consumer,
            info=logger.info, error=logger.error, task_reserved=task_reserved,
            to_system_tz=timezone.to_system, bytes=bytes,
            proto1_to_proto2=proto1_to_proto2):
    """Default task execution strategy."""
    ...
    # task event related
    # (optimized to avoid calling request.send_event)
    handle = consumer.on_task_request
    ...
    Request = symbol_by_name(task.Request)
    Req = create_request_cls(Request, task, consumer.pool, hostname, eventer)

    revoked_tasks = consumer.controller.state.revoked

    def task_message_handler(message, body, ack, reject, callbacks,
                             to_timestamp=to_timestamp):
        ....
        req = Req(
            message,
            on_ack=ack, on_reject=reject, app=app, hostname=hostname,
            eventer=eventer, task=task, connection_errors=connection_errors,
            body=body, headers=headers, decoded=decoded, utc=utc,
        )
        ...
        info('Received task: %s', req)
        ...
        handle(req)
    return task_message_handler
```

**default策略主要做了下面2件事:**

- 创建请求对象Request
- 使用handle处理request对象

Request对象的执行是**调用pool的执行方法**, 这样远程的任务请求就派发给Pool进行执行

```python
def execute_using_pool(self, pool, **kwargs):
    """Used by the worker to send this task to the pool."""
    result = pool.apply_async(
            trace_task_ret,
            args=(self._type, task_id, self._request_dict, self._body,
                  self._content_type, self._content_encoding),
            accept_callback=self.on_accepted,
            timeout_callback=self.on_timeout,
            callback=self.on_success,
            error_callback=self.on_failure,
            soft_timeout=soft_time_limit or task.soft_time_limit,
            timeout=time_limit or task.time_limit,
            correlation_id=task_id,
        )
        # cannot create weakref to None
        self._apply_result = maybe(ref, result)
    return result
```

### 调度策略

- 通过默认的路由策略直接将任务发给默认的 Queue 再由默认的 woker 处理该任务
- 默认单播（默认是单播，即一种路由策略，对应一个 Queue 和一个 worker）

- 超时控制
    - 在接入/配置异步任务的接口中暴露了超时控制的参数，可由调用者自行控制，默认没有超时机制

- 允许用户自定义调度策略，通过定义消息队列以及相应的路由策略进行接入，用户可以自定义接入广播，组播策略、以及优先级策略

```python
from kombu import Exchange, Queue

default_exchange = Exchange("default", type="direct")

INSERT_DATA_ROUTING = {"queue": "insert_data", "routing_key": "insert_data_key"}
PULL_ROUTING = {"queue": "push", "routing_key": "push_key"}
PRIORITY_ROUTING = {"queue": "pipeline_priority", "routing_key": "pipeline_push_priority"}

CELERY_ROUTES = {
    # insert_data
    "home_application.tasks.insert_data_task": INSERT_DATA_ROUTING,
    # insert_multi_data
    "home_application.tasks.insert_multi_data_task": INSERT_DATA_ROUTING,
    # push_tracking
    "home_application.tasks.push_tracking": PULL_ROUTING,
    # push_tracking
    "home_application.tasks.push_weight": PULL_ROUTING,
  	# PRIORITY
  	"home_application.tasks.priority_task": PRIORITY_ROUTING,
}

CELERY_QUEUES = (
    # keep old queue to process message left in broker, remove on next version
    Queue("default", default_exchange, routing_key="default"),
    Queue("insert_data", default_exchange, routing_key="insert_data_key"),
    Queue("push", default_exchange, routing_key="push_key"),
  	# priority queues
    Queue(
        "pipeline_priority",
        default_exchange,
        routing_key="pipeline_push_priority",
        queue_arguments={"x-max-priority": 9},
    ),
)

CELERY_DEFAULT_QUEUE = "default"  # 默认队列
CELERY_DEFAULT_EXCHANGE = "default"  # 默认 exchange
CELERY_DEFAULT_ROUTING_KEY = "default"  # 默认routing key
```

为了提高执行效率，还可以通过配置 **配置文件 `/etc/supervisord.conf`** 启动多个不同的 woker 处理不同 Queue 的任务 (本质就是通过 **Supervistor进程管理** 工具托管每个 worker的正常存活)

```shell
# default 队列 woker 配置启动 (默认的并发启动模式prefork)
[program: {{.app_code}}_celery]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -n {{.node_name}}_{{.app_code}} -c 4 -Ofair -l INFO  --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

[program: {{.app_code}}_celery_home_applications]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q push -n {{.node_name}}_{{.app_code}}_comsumer -l INFO -c 4 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

# client id 为 1 时的 push_weight_retry_1 队列配置的woker 启动
[program: {{.app_code}}_celery_home_applications]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q push_weight_retry_1 -n {{.node_name}}_{{.app_code}}_comsumer -l INFO -c 4 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

# client id 为 1 时的 push_tracking_retry_1 队列配置的woker 启动
[program: {{.app_code}}_celery_home_applications]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q push_tracking_retry_1 -n {{.node_name}}_{{.app_code}}_comsumer -l INFO -c 4 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}
 
# client id 为 2 时的 push_weight_retry_2 队列配置的woker 启动
[program: {{.app_code}}_celery_home_applications]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q push_weight_retry_2 -n {{.node_name}}_{{.app_code}}_comsumer -l INFO -c 4 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}
 
# client id 为 2 时的 push_tracking_retry_2 队列配置的woker 启动
[program: {{.app_code}}_celery_home_applications]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q push_tracking_retry_2 -n {{.node_name}}_{{.app_code}}_comsumer -l INFO -c 4 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

# 协程 并发方式启动 执行io密集型任务的 worker
[program: {{.app_code}}_celery_pipeline1]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q insert_data -n {{.node_name}}_{{.app_code}}_pipeline1 -l INFO -P gevent -c 128 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

[program: {{.app_code}}_celery_pipeline1]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/manage.py celery worker -Q pipeline_priority -n {{.node_name}}_{{.app_code}}_pipeline1 -l INFO -P gevent -c 128 --maxtasksperchild=128
directory = {{.app_container_path}}code/
stdout_logfile = {{.app_container_path}}logs/{{.app_code}}/celery.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}

# flower 进程守护
[program: {{.app_code}}_flower]
command = /cache/.bk/env/bin/python {{.app_container_path}}code/flowerd.py --persistent=true --port=6666 --url_prefix=flower --basic_auth=admin:admin --broker_api=http://guest:guest@localhost:15672/api/
directory = {{.app_container_path}}code/
stdout_logfile ={{.app_container_path}}logs/{{.app_code}}/flower.log
redirect_stderr = true
stopwaitsecs = 10
autorestart = true
environment = {{.environment}}
```



### 监控

- #### flower （https://flower-docs-cn.readthedocs.io/zh/latest/）

  ![image-20221224210804107](docs/image-20221224210804107.png)

![image-20221224210831022](docs/image-20221224210831022.png)



### 数据存储

- 如果想在 Django web 框架数据库中保存任务执行结果，还需要安装 django-celery 库（或者使用 SQLAlchemy 结果存储后端）。

    - django-celery 库基于 Django ORM和缓存框架实现了结果存储后端.

      为了在项目中使用该扩展，遵循如下四步：

        1. 安装 django-celery 库

           ```shell
           pip install django-celery
           ```

        2. 把 djcelery 加到 INSTALLED_APPS 中

           ```python
           INSTALLED_APPS = (
               'django.contrib.admin',
               'django.contrib.auth',
               'djcelery', #此处是新加入的djcelery
               'app',
           )
           ```

        3. 创建 celery 用到的数据库表

           当使用数据库作为结果存储后端时，这一步会创建用来保存任务结果的相关数据库表，以及周期任务调度器需要使用的数据库表。如果不使用周期任务和任务结果，可以跳过这一步。

           如果使用 [south](https://pypi.org/project/South/) 来做模式迁移，执行：

           ```shell
           python manage.py migrate djcelery
           ```

           如果不使用 [south](https://pypi.org/project/South/)，执行：

           ```shell
           python manage.py syncdb
           ```

        4. 配置 celery 使用 django-celery 结果存储后端

           ```python
           # 对于数据库后端，使用：
           
            app.conf.update(
            CELERY_RESULT_BACKEND='djcelery.backends.database:DatabaseBackend',
            )
           
           
           #v对于缓存后端，使用：
           
            app.conf.update(
            CELERY_RESULT_BACKEND='djcelery.backends.cache:CacheBackend',
           
           ```

           如果你将 Django settings 作为 Celery 的配置来源，可以直接在 settings 模块中加上 CELERY_RESULT_BACKEND 配置项，不需要 app.conf.update

- 如果我们需要跟踪任务状态，就要将Celery的结果保存下来

    - 可以用`SQLAlchemy`、`Redis`、`RabbitMQ/AMQP`等来进行结果的存储

    - 例子使用`redis`来进行存储，通过`backend`来进行配置存储的地方，修改`tasks.py`如下

      ```python
      from celery import Celery
      
      # 我们这里案例使用redis作为broker
      app = Celery(
          'demo',
          broker='redis://127.0.0.1:6379/5',
          backend='redis://127.0.0.1:6379/6'
      )
      
      
      # 创建任务函数
      @app.task
      def my_task():
          print("任务函数正在执行....")
      
      
      ```

- 是否支持分库分表

    - 不支持



#### 认证和权限

- Celery 没有认证和权限，但是例如查询任务执行状态可以通过代码以及flower监控平台