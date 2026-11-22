# Python 异步编程入门：asyncio 核心机制与踩坑避坑指南.md
## 前言
在处理大量网络请求、数据库读写这类IO密集场景时，传统多线程方案会遇到线程数量上限、操作系统线程调度开销大的问题。当并发请求达到几千、上万级别，创建大量线程会造成内存暴涨、上下文切换频繁，系统性能快速下降。

Python 提供的 `asyncio` 异步编程方案，采用**单线程、协程调度**模型，在用户态完成任务切换，不需要操作系统创建真实线程，协程占用资源极小。单进程就可以轻松支撑上万并发IO任务。

很多初学者接触 asyncio 很容易混淆协程、任务、事件循环，并且写出“假异步”代码：写了async/await，但程序依旧串行执行，完全没有并发效果。
本文从底层核心机制入手，搭配可运行示例，讲解 asyncio 基础概念、执行流程，同时整理开发高频踩坑点与避坑方案，帮你快速落地异步项目。

> 重要前置：asyncio 擅长 **IO密集型任务**；**CPU密集型任务不适合asyncio**，长时间同步计算会阻塞整个事件循环。

## 一、asyncio 核心基础概念
### 1. 协程（coroutine）
使用 `async def` 定义的函数，就是协程函数。
调用协程函数**不会立刻执行函数内部代码**，而是返回一个协程对象，类似生成器，需要交给事件循环调度运行。
`await` 关键字：只能写在 `async def` 函数内部，用来挂起当前协程，让出事件循环，等待可等待对象完成。

可等待对象（awaitable）分为三类：
1. 协程（coroutine）
2. 任务（Task）：对协程的封装，交给事件循环并发调度
3. Future：底层占位对象，代表一个未来完成操作的结果，业务代码一般很少直接使用

### 2. 事件循环（event loop）
事件循环是 asyncio 的核心调度器，相当于协程的“管家”。
职责：
- 注册、调度协程/任务
- IO事件监听
- 协程挂起、恢复执行
- 处理任务回调

在 Python3.7+，推荐使用高层API `asyncio.run()`，自动创建、关闭事件循环，不再需要手动管理 `get_event_loop()`。

### 3. Task 任务
`asyncio.create_task()` 将协程包装成Task，提交给事件循环后台调度，实现并发。
> 关键点：**只有创建Task，协程才会并发执行；单纯连续await，代码依旧串行。**

## 二、最简示例：串行 vs 并发对比
### 串行版本（没有并发）
```python
import asyncio
import time

async def demo_task(name, delay):
    print(f"{name} 开始执行，等待 {delay}s")
    await asyncio.sleep(delay)
    print(f"{name} 执行完成")

async def main():
    start = time.time()
    # 依次await，串行执行
    await demo_task("任务A", 2)
    await demo_task("任务B", 2)
    print(f"总耗时：{time.time() - start:.2f}s")

if __name__ == "__main__":
    asyncio.run(main())
```
运行结果：总耗时约4秒。两个任务排队执行。

### 并发版本（create_task）
```python
import asyncio
import time

async def demo_task(name, delay):
    print(f"{name} 开始执行，等待 {delay}s")
    await asyncio.sleep(delay)
    print(f"{name} 执行完成")

async def main():
    start = time.time()
    # 创建任务，提交到事件循环并发调度
    task1 = asyncio.create_task(demo_task("任务A", 2))
    task2 = asyncio.create_task(demo_task("任务B", 2))
    await task1
    await task2
    print(f"总耗时：{time.time() - start:.2f}s")

if __name__ == "__main__":
    asyncio.run(main())
```
运行结果：总耗时约2秒，任务并发执行。

### 批量并发：asyncio.gather
当需要一次性批量运行大量协程，收集所有返回结果，优先使用 `asyncio.gather()`。
```python
import asyncio
import time

async def demo_task(name, delay):
    print(f"{name} 开始执行，等待 {delay}s")
    await asyncio.sleep(delay)
    return f"{name} 结果"

async def main():
    start = time.time()
    tasks = [
        demo_task("任务A", 2),
        demo_task("任务B", 2),
        demo_task("任务C", 2)
    ]
    # 批量并发，收集返回值
    res = await asyncio.gather(*tasks)
    print(res)
    print(f"总耗时：{time.time() - start:.2f}s")

if __name__ == "__main__":
    asyncio.run(main())
```
`asyncio.gather` 特点：
- 默认任意一个任务抛出异常，整体直接抛出异常；
- 设置 `return_exceptions=True` 可以把异常当作返回值返回，不打断整体批量任务。

## 三、事件循环调度原理
协程属于**协作式调度**，和线程抢占式调度完全不同：
- 线程：操作系统强制时间片抢占切换；
- asyncio协程：**只有当前协程执行到await主动让出控制权时，事件循环才会调度其他协程运行**。

如果协程内部出现同步阻塞代码（time.sleep()、同步requests请求、大量同步循环计算），**不会主动让出控制权，事件循环被卡死，所有其他协程全部暂停**。
这是asyncio最核心的坑。

✅ 异步等待：`await asyncio.sleep()` → 挂起协程，让出事件循环
❌ 同步阻塞：`time.sleep()` → 阻塞整个事件循环，其他协程无法运行

## 四、asyncio 高频踩坑与避坑指南
### 坑1：async函数不加await，直接调用
```python
async def hello():
    print("hello")

# 错误写法：只生成协程对象，函数不会执行
hello()
```
警告提示：`RuntimeWarning: coroutine 'hello' was never awaited`
> 解决方案：协程对象必须交给事件循环，使用await或者包装为Task。

### 坑2：混用同步IO库（requests）
requests是同步HTTP库，在async函数内部直接调用requests.get()，会阻塞事件循环。
❌ 错误示例
```python
import asyncio
import requests

async def fetch():
    # requests同步请求，阻塞事件循环！
    resp = requests.get("[https://www.baidu.com](https://www.baidu.com)")
    print(resp.status_code)

async def main():
    task1 = asyncio.create_task(fetch())
    task2 = asyncio.create_task(fetch())
    await task1
    await task2

asyncio.run(main())
```
✅ 正确方案：使用异步HTTP客户端 aiohttp
```python
import asyncio
import aiohttp

async def fetch(session):
    async with session.get("[https://www.baidu.com](https://www.baidu.com)") as resp:
        print(resp.status)

async def main():
    async with aiohttp.ClientSession() as session:
        task1 = asyncio.create_task(fetch(session))
        task2 = asyncio.create_task(fetch(session))
        await task1
        await task2

asyncio.run(main())
```

> 补充：如果业务必须使用同步库，可以用 `asyncio.to_thread()` 将同步代码放到独立线程执行，不阻塞事件循环。适合少量同步代码，不建议大规模使用。
```python
# 将同步函数丢进线程执行
result = await asyncio.to_thread(requests.get, "[https://www.baidu.com](https://www.baidu.com)")
```

### 坑3：asyncio.gather 一个任务异常，整体崩溃
默认情况下，gather只要有一个协程抛出异常，gather立刻抛出异常，剩余任务依旧会继续执行，但结果无法收集。

✅ 解决方案：开启 `return_exceptions=True`
```python
res_list = await asyncio.gather(*tasks, return_exceptions=True)
# 遍历结果，判断每个返回值是否是异常
for res in res_list:
    if isinstance(res, Exception):
        print("任务失败：", res)
    else:
        print("任务成功结果：", res)
```

### 坑4：忘记限制并发数量，瞬间创建上万任务，打爆服务
直接使用gather一次性提交成千上万个网络请求，会瞬间建立大量连接，目标服务限流、断开连接，本地文件描述符耗尽。

✅ 解决方案：使用 `asyncio.Semaphore` 信号量限制并发数
```python
import asyncio
import aiohttp

# 限制最大并发为3
sem = asyncio.Semaphore(3)

async def fetch(session, url):
    async with sem:  # 进入信号量，控制并发
        async with session.get(url) as resp:
            return resp.status

async def main():
    urls = ["[https://www.baidu.com](https://www.baidu.com)"] * 10
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        print(results)

asyncio.run(main())
```

### 坑5：嵌套事件循环，在已经运行的事件循环内部调用 asyncio.run()
常见场景：jupyter notebook、FastAPI内部，代码已经处于事件循环中，再次调用`asyncio.run()`直接报错。
> Python3.7+：`asyncio.run()` 会新建事件循环，不允许嵌套。
> 解决方案：区分运行环境，notebook使用 `await` 直接执行协程；生产项目避免嵌套调用。

### 坑6：长时间CPU计算阻塞事件循环
```python
async def heavy_work():
    # 纯CPU密集循环，没有await，阻塞事件循环
    s = 0
    for i in range(100000000):
        s += i
    return s
```
✅ 方案：
1. 把CPU heavy逻辑丢进多进程：`asyncio.to_process`（新版python）或者 multiprocessing；
2. 循环内部手动插入短暂await，让出事件循环（仅缓解，不推荐大规模计算）。

### 坑7：任务异常没有捕获，静默丢失异常
Task内部抛出异常，如果没有await，也没有增加异常回调，程序不会立刻报错，异常会被隐藏，很难排查。
```python
import asyncio

async def error_task():
    raise ValueError("任务内部出错")

async def main():
    # 创建任务，但没有await，异常不会立刻抛出
    t = asyncio.create_task(error_task())
    await asyncio.sleep(1)

asyncio.run(main())
```
> 避坑：要么await任务捕获异常；或者增加add_done_callback捕获异常。
```python
def handle_task_result(task: asyncio.Task):
    if exc := task.exception():
        print(f"任务异常：{exc}")

async def main():
    t = asyncio.create_task(error_task())
    t.add_done_callback(handle_task_result)
    await asyncio.sleep(1)
```

## 五、常用工具函数选型总结
| API | 用途 |
| ---- | ---- |
| asyncio.create_task | 创建后台任务，并发执行协程 |
| asyncio.gather | 批量执行多个协程，收集全部返回值 |
| asyncio.wait | 等待任务集合，可设置等待完成条件（FIRST_COMPLETED） |
| asyncio.Semaphore | 限制最大并发数量，防请求风暴 |
| asyncio.to_thread | 在独立线程运行同步代码，防止阻塞事件循环 |
| asyncio.sleep | 异步休眠，让出事件循环，不要用time.sleep |

## 六、asyncio 适用场景与不适用场景
✅ 适合场景：
- 大量HTTP接口爬虫、API批量调用
- 异步数据库（asyncpg、aiomysql）读写
- WebSocket长连接服务
- 消息队列异步消费

❌ 不适合场景：
- 大量CPU密集计算（加密、循环运算、图像处理）
- 大量同步第三方库（没有异步SDK）

## 七、总结
asyncio异步编程的核心本质：**协作式调度，遇到await主动让出事件循环，实现IO并发。**
使用asyncio开发，记住三条铁律：
1. 异步代码必须搭配异步IO库，禁止在协程内部使用同步阻塞IO；
2. 并发需要显式创建Task或者使用gather，连续await只会串行；
3. 控制并发上限，做好异常捕获，防止任务异常静默丢失。

很多初学者写出来的异步代码，性能和串行代码没有差别，根源就是没有理解协程调度机制，混用同步阻塞代码。只要避开上面这些高频坑，asyncio就可以轻松支撑高并发IO业务。







https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/about
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/learn
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/favorited
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/group
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/following
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/about
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/learn
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/favorited
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/group
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/following
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/about
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/learn
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/favorited
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/group
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/following
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/about
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/learn
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/favorited
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/group
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/following
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/about
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/learn
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/favorited
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/group
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/following
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/about
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/learn
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/favorited
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/group
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/following
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/about
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/learn
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/favorited
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/group
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/following
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/about
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/learn
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/favorited
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/group
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/following
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/about
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/learn
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/favorited
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/group
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/following
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/about
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/learn
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/favorited
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/group
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/following
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/about
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/learn
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/favorited
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/group
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/following
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/about
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/learn
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/favorited
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/group
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/following
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/about
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/learn
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/favorited
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/group
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/following
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/about
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/learn
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/favorited
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/group
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/following
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/about
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/learn
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579/favorited
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d/group
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73/following
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face/about
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36/learn
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121/favorited
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400/group
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5/following
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f/about
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026/learn
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea/favorited
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89/group
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931/following
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa/about
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa/learn
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57/favorited
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef/group
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789/following
https://yuanzhuo.bnu.edu.cn/user/5c623e472e39f348655368ed6c6df501cb194579
https://yuanzhuo.bnu.edu.cn/user/7f4a0532d28294382f469e9e5d1f8104c76d799d
https://yuanzhuo.bnu.edu.cn/user/20bd5e9b46f0d7a8f6b4753da8a4f70d59ac6a73
https://yuanzhuo.bnu.edu.cn/user/fece4199b016fb75d8e91b8ef1974f36dd54face
https://yuanzhuo.bnu.edu.cn/user/f7773eeed1f8d3425e7b5ed700a1bb6dcde73c36
https://yuanzhuo.bnu.edu.cn/user/6fb4b548a8c5a157b65acecfd279d19b9b5d7121
https://yuanzhuo.bnu.edu.cn/user/aa7e7a417f972a433169760b1a9e444f07081400
https://yuanzhuo.bnu.edu.cn/user/958798d85500a2ae24cd2203b1ab18c96d4e7fd5
https://yuanzhuo.bnu.edu.cn/user/9e2660d21271ac0bd0ab141e330f0258b8efea7f
https://yuanzhuo.bnu.edu.cn/user/ff0a231c28cfa5b689645b8a76cabcbb1002b026
https://yuanzhuo.bnu.edu.cn/user/8fcb65bac7673ea51f15774231cfedaf68aa7bea
https://yuanzhuo.bnu.edu.cn/user/8d6e16b2520f82ad9adb53c3f68d15151d360b89
https://yuanzhuo.bnu.edu.cn/user/92aeac6b47c55e3003dfb088f8fce4c3481be931
https://yuanzhuo.bnu.edu.cn/user/597f62f6c26e5d52055030624da88bf9a4f90eaa
https://yuanzhuo.bnu.edu.cn/user/eb719e5e16299407170171d596090a7a0f5326fa
https://yuanzhuo.bnu.edu.cn/user/be5b7f5c8965d8a44945ef774f81a81c4c7edc57
https://yuanzhuo.bnu.edu.cn/user/ddfbdaf6e77588301332e5c008d66b0cf1458aef
https://yuanzhuo.bnu.edu.cn/user/8018ab5d2208bf7961bc5c924643f28ff1cbb789
