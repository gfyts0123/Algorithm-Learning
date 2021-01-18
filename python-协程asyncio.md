# python-协程asyncio

## 关键内容纵览

- **event-loop**：事件（无限）循环
- **coroutine**：协程对象
- **task**：任务是对协程的封装，包含状态等其他信息
- **future**：表示将来执行或没有执行的任务的结果，和task（future子类）没有本质区别
- **async/await**：协程关键字

## 实例说明

### 协程的定义与使用

```python
import time
import asyncio

async def do_some_work(x):
    print('Waiting: ', x)

now = lambda : time.time()
start = now()
coroutine = do_some_work(2)     # 以async函数注册为协程对象

loop = asyncio.get_event_loop()     # 事件循环

# 在这里已经将coroutine注册，之后不能再直接使用coroutine
# task = asyncio.ensure_future(coroutine)
task =  loop.create_task(coroutine)
print(task)     # <Task pending coro=<do_some_work() running

# 如果传进未注册的coroutine或future，会自动封装为task
loop.run_until_complete(task)          
print(task)     # <Task finished coro=<do_some_work() done
print('TIME: ', now() - start)
```

`asyncio.ensure_future` 和 `loop.create_task`在此处效果是相同的：区别为直接注册为task/先创建future对象

### 回调与执行结果

```python
import time
import asyncio

async def do_some_work(x):
    print('Waiting: ', x)
    return 'Get callback {}'.format(x)

def callback(future):
    print('Callback: ', future.result())

now = lambda : time.time()
start = now()
coroutine = do_some_work(0.02)     # 以async函数注册为协程对象

loop = asyncio.get_event_loop()     # 事件循环
task = asyncio.ensure_future(coroutine)

# 注册异步回调函数
task.add_done_callback(callback)    

loop.run_until_complete(task)     # 开始运行
# 不需要注册回调，同步返回结果
# print(task.result())    
print('TIME: ', now() - start)
```

- 一种方式为：`task.add_done_callback`，异步回调
- 另一种方式为：`task.result()`同步地获取结果（相对来说更简单）

### await与协程并发

```python
import time
import asyncio

async def do_some_work(x):
    print('Waiting: ', x)
    await asyncio.sleep(x)      
    return 'Get callback {}'.format(x)

now = lambda : time.time()
start = now()
coroutine1 = do_some_work(1)
coroutine2 = do_some_work(2)
coroutine3 = do_some_work(4)

tasks = [
    asyncio.ensure_future(coroutine1),
    asyncio.ensure_future(coroutine2),
    asyncio.ensure_future(coroutine3)
]
loop = asyncio.get_event_loop()
# asyncio.wait(tasks)接受task列表，asyncio.gather(*tasks)接受一堆task
loop.run_until_complete(asyncio.wait(tasks))
for task in tasks:
    print(task.result())
# 我们同步需要等待7秒，那么协程需要几秒呢？
print('TOTAL TIME: ', now() - start)
```

- 这样就实现了**并发**，什么是并发？可以形象理解为**一个人在一段时间内做多个事情**；与此相对的是**并行**，并行为**好几个人在一段时间做多个事情**，比如分一个馒头一起吃，吃一个算一个task，一段时间要吃多个馒头。
- 耗时的IO被并发解决，提升了整体的效率，最后协程只需要约4秒的时间。

### 协程嵌套

```python
import time
import asyncio

async def do_some_work(x):
    print('Waiting: ', x)
    await asyncio.sleep(x)      
    return 'Get callback {}'.format(x)

async def main():
    coroutine1 = do_some_work(1)
    coroutine2 = do_some_work(2)
    coroutine3 = do_some_work(4)

    tasks = [
        asyncio.ensure_future(coroutine1),
        asyncio.ensure_future(coroutine2),
        asyncio.ensure_future(coroutine3)
    ]
    # 挂起执行tasks，直接返回最后结果，也可以使用asyncio.wait/asyncio.as_completed
    return await asyncio.gather(*tasks)

start = time.time()
loop = asyncio.get_event_loop()
# 这里将main函数挂入循环，获取到执行结果并解析
results = loop.run_until_complete(main())
for result in results:
    print(result)

print('TOTAL TIME: ', time.time() - start)
```

- 在此处我们加入了`async main`作为主函数，在一个协程中await另外的协程实现嵌套
- 协程嵌套的方法十分灵活，可以在main里直接处理，可以抛出，可以用不同方法等，根据实际需要进行选择。

### 协程ctrl+c停止

```python
import time
import asyncio

async def do_some_work(x):
    print('Waiting: ', x)
    await asyncio.sleep(x)      
    return 'Get callback {}'.format(x)

async def main():
    coroutine1 = do_some_work(1)
    coroutine2 = do_some_work(2)
    coroutine3 = do_some_work(4)

    tasks = [
        asyncio.ensure_future(coroutine1),
        asyncio.ensure_future(coroutine2),
        asyncio.ensure_future(coroutine3)
    ]
    return await asyncio.gather(*tasks)

start = time.time()
loop = asyncio.get_event_loop()
# 这里将main函数挂入循环，获取到执行结果并解析
try:
    results = loop.run_until_complete(main())
# ctrl+c取消的处理
except KeyboardInterrupt as e:
    print(asyncio.Task.all_tasks())
    print(asyncio.gather(*asyncio.Task.all_tasks()).cancel())
    loop.stop()
    loop.run_forever()      # 重启loop，此时没有task，所以循环会结束。
finally:
    loop.close()
# for result in results:
    # print(result)

print('TOTAL TIME: ', time.time() - start)
```

可以看到，只需要在对应位置try-except `KeyboardInterrupt`即可

### 线程与协程

```python
from threading import Thread
import asyncio
import time

# 用于子线程设置loop：loop非线程安全，只能在一个线程里调度任务
def start_loop(loop):
    asyncio.set_event_loop(loop)
    loop.run_forever()

async def do_some_work(x):
    print('Waiting {}'.format(x))
    await asyncio.sleep(x)
    print('Do work {}'.format(x))

def more_work(x):
    print('More work {}'.format(x))
    time.sleep(x)
    print('Finished more work {}'.format(x))

start = time.time()
new_loop = asyncio.new_event_loop()
# 新起线程运行事件循环, 防止阻塞主线程
t = Thread(target=start_loop, args=(new_loop,))
t.start()
print('TIME: {}'.format(time.time() - start))

# time.sleep()是同步阻塞的，按顺序同步执行
# call_soon_threadsafe线程安全，不会有共享变量问题
# 注意：此new_loop为线程t的loop（让子线程的loop调度这两个任务）
# new_loop.call_soon_threadsafe(more_work, 6)
# new_loop.call_soon_threadsafe(more_work, 3)

# 异步执行（让子线程的loop调度这两个任务）
asyncio.run_coroutine_threadsafe(do_some_work(6), new_loop)
asyncio.run_coroutine_threadsafe(do_some_work(4), new_loop)
```

- loop非线程安全，只能在一个线程里调度任务，在这里我们新建了一个子线程用于调度任务。`run_coroutine_threadsafe`可以实现并发。

### master-worker主从模式

- 协程更适合单线程操作，以master主线程监听，子线程处理的方式
- 可以将子线程设置为主线程的守护线程，主线程结束时子线程也退出

参考链接：[@人世间](https://www.jianshu.com/p/b5e347b3a17c)