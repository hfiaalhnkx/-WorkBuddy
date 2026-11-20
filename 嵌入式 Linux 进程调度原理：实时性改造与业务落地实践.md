# 嵌入式 Linux 进程调度原理：实时性改造与业务落地实践.md

## 前言

在嵌入式Linux开发中，很多开发者会混淆“响应快”和“实时”。普通桌面Linux的CFS调度追求所有进程公平分享CPU，适合吞吐量优先的场景；而工控采集、运动控制、音视频解码、传感器数据闭环这类嵌入式业务，要求**任务在确定时间内得到响应**，时延抖动必须控制在可接受范围，这就是实时性需求。

嵌入式设备硬件资源普遍受限：单核/弱多核SoC、低速Flash、内存紧张，中断、后台日志、看门狗、文件系统IO都会抢占业务进程，引入不可预测的时延。单纯依靠应用层编码优化，很难解决调度层面带来的抖动。想要实现低延迟、低抖动，就必须吃透Linux内核调度底层原理，再结合业务场景做调度策略、CPU隔离、优先级、中断优化等一系列实时改造。

本文从调度器底层原理入手，对比普通调度与实时调度差异，讲解实时调度相关API、命令实操，同时梳理嵌入式产品落地的改造方案、风险点、性能验证方法与常见踩坑案例。适合工控、机器人、边缘网关、智能硬件方向的嵌入式开发人员阅读。

## 一、Linux 进程调度基础模型

Linux内核调度器负责从就绪队列挑选进程投入CPU运行，核心目标分为两类：**公平性** 和 **实时确定性**。

### 1. CFS 完全公平调度器（普通进程默认）

CFS（Completely Fair Scheduler）是Linux默认调度器，用于非实时普通进程，嵌入式Linux默认配置下所有应用进程都跑在CFS。
CFS不再使用传统固定时间片，引入**虚拟运行时间(vruntime)**。内核维护一棵红黑树，就绪进程按照vruntime排序，vruntime最小的进程优先获得CPU。

- nice值：取值范围 `-20 ~ 19`，nice越小，权重越高，分到的CPU时间越多。
- 动态优先级：进程长时间休眠（IO阻塞）会获得优先级奖励，唤醒后可以优先运行；持续占用CPU的进程会被惩罚。

**嵌入式场景CFS的短板**
CFS的设计目标是公平，不保证时延确定性。当系统负载高时，后台进程、日志进程、shell脚本、内核线程都会抢占业务任务，造成业务处理时延抖动。哪怕业务进程优先级调高，依然会被其他就绪进程抢占，不适合硬实时闭环控制业务。

### 2. Linux 实时调度类（SCHED_FIFO / SCHED_RR）

实时调度类优先级**高于CFS普通进程**，实时进程优先级范围 `1~99`，数字越大优先级越高。只要有高优先级实时进程处于就绪态，就会抢占所有低优先级任务（包括普通CFS进程）。

1. **SCHED_FIFO（先进先出实时调度）**
SCHED_FIFO没有时间片。一旦高优先级FIFO进程获得CPU，会持续运行，直到：

- 主动阻塞（sleep、等待信号量、IO）；
- 主动调用sched_yield()让出CPU；
- 被更高优先级实时进程抢占。

> 
> 适用场景：短周期、高优先级闭环控制任务，例如电机控制、传感器实时采样。
> ⚠️风险：如果SCHED_FIFO进程出现死循环，没有阻塞逻辑，会独占CPU，系统直接卡死，看门狗复位。

2. **SCHED_RR（实时轮询调度）**
SCHED_RR属于实时调度，同优先级的多个RR进程之间按照固定时间片轮流执行。不同优先级依然遵循高优先级优先抢占。> 
> 适用场景：多个同等优先级的实时任务，需要互相分配CPU时间，防止单个任务无限占用。

> 
> 重要区分：SCHED_FIFO和SCHED_RR都属于**软实时**，标准Linux内核（非PREEMPT_RT补丁）不是硬实时系统，只能降低抖动，无法做到绝对确定性时延。

### 3. 调度抢占模型：影响实时性的核心开关

Linux内核有多种抢占模式，直接决定内核态代码是否可以被实时进程抢占，是嵌入式实时改造的关键点：

1. **No Preempt（不抢占）**：内核执行系统调用时不可抢占。只有从内核态返回用户态时才做进程调度。时延抖动大，一般不用于实时业务。
2. **Voluntary Preempt（自愿抢占）**：内核代码主动插入抢占点，降低最大延迟。
3. **Preemptible Kernel（可抢占内核）**：大部分内核代码可以被抢占，中断返回时可以调度。
4. **PREEMPT_RT 实时补丁**：把自旋锁改成可睡眠的互斥锁，把中断处理线程化，大幅降低内核最大延迟，是嵌入式硬实时场景最常用方案。> 
> 注意：PREEMPT_RT不是主线内核默认配置，需要单独打补丁、重新编译内核，很多商用嵌入式SoC厂商会提供预编译RT内核固件。

## 二、调度策略修改：API与命令行实战

### 1. chrt 命令：运行时修改调度策略

嵌入式调试阶段，经常使用chrt命令修改进程调度属性，无需重新编译应用。

```
# 设置PID=1001进程为SCHED_FIFO，优先级50
chrt -f -p 50 1001

# 设置为SCHED_RR，优先级30
chrt -r -p 30 1001

# 查看进程当前调度策略和优先级
chrt -p 1001
```

> 
> 权限说明：修改为实时调度策略需要root权限，嵌入式产品一般业务进程以root运行，或者配置capabilities能力，不用全开root。

### 2. 应用代码 sched_setscheduler API

产品正式代码，推荐在应用初始化阶段直接调用`sched_setscheduler`设置调度策略，不需要依赖外部chrt脚本。

```
#include <stdio.h>
#include <sched.h>
#include <unistd.h>

int set_sched_fifo(int pid, int priority)
{
    struct sched_param param;
    param.sched_priority = priority;
    // 设置SCHED_FIFO
    if(sched_setscheduler(pid, SCHED_FIFO, &param) < 0){
        perror("sched_setscheduler failed");
        return -1;
    }
    return 0;
}

int main(void)
{
    printf("设置当前进程为SCHED_FIFO，优先级40\n");
    set_sched_fifo(getpid(), 40);
    while(1){
        // 实时业务循环
        usleep(1000);
    }
    return 0;
}
```

### 3. CPU亲和性（CPU隔离）

多核嵌入式SoC，可以通过`taskset`或者`sched_setaffinity`把实时业务进程绑定到指定CPU核心，实现**CPU隔离**：后台进程、内核线程、日志、文件系统全部跑在其他核心，实时任务独占一个核，减少核间抢占、缓存失效带来的抖动。

命令行示例：

```
# 将进程1001绑定到CPU核心0
taskset -p 0x01 1001
```

C代码设置CPU亲和性：

```
cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(0, &mask); // 绑定CPU0
if(sched_setaffinity(getpid(), sizeof(mask), &mask) <0){
    perror("sched_setaffinity");
}
```

> 
> 产品实践：配合内核启动参数 `isolcpus=0`，把CPU0隔离出来，内核不会在该核心自动调度普通进程，专门留给实时业务任务。

## 三、嵌入式Linux实时性完整改造方案（业务落地）

单纯修改进程调度策略远远不够。系统内中断、内核线程、文件IO、内存分配都会引入延迟，完整实时改造是一套组合方案。

### 1. 内核配置层面改造

1. 选择合适抢占模型：实时业务优先使用PREEMPT_RT内核；普通低抖动场景可选用可抢占内核。
2. 关闭不必要内核功能：关闭调试、trace、内核统计、冗余驱动，减少内核后台开销。
3. 中断优化：将高频率中断迁移到非实时核心；中断线程化（RT补丁特性），避免中断长时间占用CPU阻塞实时任务。
4. 关闭CPU节能调频：CPU降频、动态调频会带来巨大时延波动，实时产品固定CPU主频。内核启动参数 `cpufreq=performance`。

### 2. 系统根文件系统与后台服务裁剪

很多嵌入式设备实时抖动来自后台服务：syslog、日志轮转、dbus、定时任务、网络服务。

- 精简根文件系统，只保留业务必需进程；
- 日志输出策略调整：实时业务禁止高频同步写Flash，日志放入内存tmpfs，异步落盘；
- 后台进程统一设置为低优先级CFS调度，禁止后台任务抢占实时核心。

### 3. 应用层编码规范（实时业务开发约束）

1. 实时任务避免使用malloc动态内存：malloc会锁，内存分配带来不确定延迟，内存预先一次性分配。
2. 避免阻塞IO：实时任务不要直接操作块设备、Flash读写；IO操作放到低优先级线程。
3. 锁的使用：实时任务优先使用轻量无锁机制；如果需要互斥，RT内核使用rt_mutex，避免普通自旋锁长时间占用CPU。
4. 任务拆分：高优先级实时任务尽量短小，只做关键采集与控制；非关键计算、数据打包、上报交给低优先级进程。

### 4. 优先级分层设计（非常关键）

产品中不能把所有任务都设置成高优先级实时，否则优先级失效，出现互相抢占。推荐分层设计示例：

- 优先级80~99：紧急硬件闭环控制（电机、运动控制）SCHED_FIFO
- 优先级50~79：传感器采集、实时数据处理 SCHED_FIFO
- 优先级1~49：音视频实时处理 SCHED_RR
- nice 0~19：日志、数据存储、网络上报、后台监控（CFS普通进程）

## 四、实时性指标测试与验证

改造完成后，需要量化时延指标，不能只靠主观感受。嵌入式常用测试工具：

1. **cyclictest**：PREEMPT_RT标配工具，测量调度延迟，统计最大、最小、平均时延，是评估实时性能最标准工具。

```
cyclictest -t1 -p 80 -n -i 10000 -l 100000
```

参数说明：单线程，优先级80，循环10万次，周期10ms，输出延迟统计。重点关注`Max`最大延迟。
2. **rt-tests** 工具集：包含hackbench、pi_test，用于测试优先级反转。
3. ftrace：内核跟踪工具，定位延迟根源：中断抢占、锁竞争、调度延迟。

典型验收标准参考（根据业务需求调整）：

- 软实时：最大抖动 < 1ms
- PREEMPT_RT硬实时场景：最大抖动控制在百微秒级别（取决于硬件）

## 五、实时调度典型坑点与故障排查

### 1. 优先级反转

低优先级任务持有锁，高优先级实时任务等待锁，此时中等优先级任务抢占CPU，导致高优先级任务长时间阻塞。
**解决方案**：使用**优先级继承互斥锁（pthread_mutexattr_setprotocol PTHREAD_PRIO_INHERIT）**，RT内核原生支持。

### 2. 实时进程死循环卡死整机

SCHED_FIFO进程代码死循环、没有阻塞点，独占CPU，所有其他任务无法运行。
对策：

- 增加看门狗监控实时业务；
- 增加超时检测逻辑；
- 多核架构下，实时核心隔离，部分内核线程保留在其他核心，保证系统还能响应调试串口。

### 3. 内存缺页异常带来抖动

实时进程访问未加载物理内存，触发缺页中断，引入不确定延迟。
解决：进程启动后使用`mlock()`/`mlockall()`锁定内存，将虚拟内存提前加载到物理内存，禁止内存换出。

```
mlockall(MCL_CURRENT | MCL_FUTURE);
```

### 4. 时钟精度问题

普通Linux定时器精度受jiffies影响，周期任务抖动大。RT内核开启高精度定时器hrtimer，满足微秒级周期任务。

## 六、总结

嵌入式Linux进程实时性改造，不是简单给进程设置SCHED_FIFO高优先级，而是一套全栈优化体系：内核抢占模型选型、中断优化、CPU隔离、后台服务裁剪、应用编码约束、锁与内存锁定，缺一不可。

CFS调度适合非实时后台业务；SCHED_FIFO/SCHED_RR用于实时任务，但是原生Linux不具备硬实时能力，对时延要求极高的场景，必须引入PREEMPT_RT补丁。同时要做好优先级分层、优先级反转防护、内存锁定，避免高优先级任务引发整机卡死。

上线前必须用cyclictest量化测试调度延迟，验证负载压力下的最大抖动。调度实时性属于底层基础能力，稳定可靠的调度体系，是工控、机器人、边缘采集等嵌入式产品长期稳定运行的基石。







https://vr.justeasy.cn/view/179cp5111p9m6771-1795196782.html
https://vr.justeasy.cn/view/17e95199b5684q99-1795196911.html
https://vr.justeasy.cn/view/m017951k9h69yu41-1795196953.html
https://vr.justeasy.cn/view/17eh951y96hev983-1795196995.html
https://vr.justeasy.cn/view/11t7895i19u70726-1795197037.html
https://vr.justeasy.cn/view/1t7w9510970vq368-1795197080.html
https://vr.justeasy.cn/view/179th5r1987s1m10-1795197122.html
https://vr.justeasy.cn/view/j1j7oi95194715w3-1795197165.html
https://vr.justeasy.cn/view/179519a271wl4i95-1795197207.html
https://vr.justeasy.cn/view/171kz9n5197j23s8-1795197250.html
https://vr.justeasy.cn/view/17c951797229u381-1795197292.html
https://vr.justeasy.cn/view/1769j50x19t7e323-1795197335.html
https://vr.justeasy.cn/view/h21kq72951973c66-1795197377.html
https://vr.justeasy.cn/view/i1795l194724x0q8-1795197420.html
https://vr.justeasy.cn/view/1769519nrn790451-1795197462.html
https://vr.justeasy.cn/view/l17951g97154kp93-1795197505.html
https://vr.justeasy.cn/view/17l9954197hw5w36-1795197547.html
https://vr.justeasy.cn/view/11795d1n9752zq78-1795197589.html
https://vr.justeasy.cn/view/16879k5t19762ju0-1795197632.html
https://vr.justeasy.cn/view/lt11795h19t76y63-1795197674.html
https://vr.justeasy.cn/view/179ww51977p9p605-1795197717.html
https://vr.justeasy.cn/view/17z951g9974hp748-1795197760.html
https://vr.justeasy.cn/view/17p95e3197k79241-1795197802.html
https://vr.justeasy.cn/view/aco1j79g51b97833-1795197845.html
https://vr.justeasy.cn/view/17p9054d1967n876-1795197887.html
https://vr.justeasy.cn/view/1m795ndxj9197918-1795197929.html
https://vr.justeasy.cn/view/hw1795197u960fi0-1795197972.html
https://vr.justeasy.cn/view/21795ds631998003-1795198014.html
https://vr.justeasy.cn/view/179oc5h198ml04q5-1795198056.html
https://vr.justeasy.cn/view/179b251fn94808j8-1795198099.html
https://vr.justeasy.cn/view/17l958o1968am131-1795198142.html
https://vr.justeasy.cn/view/187709p51968w173-1795198185.html
https://vr.justeasy.cn/view/413u7956129e8216-1795198227.html
https://vr.justeasy.cn/view/l1wn779051984258-1795198269.html
https://vr.justeasy.cn/view/10z7951997c83i01-1795198312.html
https://vr.justeasy.cn/view/179151zi31998343-1795198355.html
https://vr.justeasy.cn/view/179d25vg19a838w6-1795198397.html
https://vr.justeasy.cn/view/m1lwrp7951d98428-1795198440.html
https://vr.justeasy.cn/view/y17951b9w2u84c70-1795198482.html
https://vr.justeasy.cn/view/1679c51f9o835v13-1795198524.html
https://vr.justeasy.cn/view/v17bim92519x8555-1795198567.html
https://vr.justeasy.cn/view/11779995198al598-1795198609.html
https://vr.justeasy.cn/view/17i9k5n1j9px8641-1795198652.html
https://vr.justeasy.cn/view/r17395ow1mv98683-1795198695.html
https://vr.justeasy.cn/view/127951y98pg7f296-1795198737.html
https://vr.justeasy.cn/view/81a79p519wt87688-1795198779.html
https://vr.justeasy.cn/view/1795c1898s871l70-1795198822.html
https://vr.justeasy.cn/view/61795uk1988pnd53-1795198864.html
https://vr.justeasy.cn/view/r1739mwf519u8896-1795198907.html
https://vr.justeasy.cn/view/179519e86re93p48-1795198949.html
https://vr.justeasy.cn/view/z179519d895582q0-1795198991.html
https://vr.justeasy.cn/view/1hz795oi199m0f22-1795199036.html
https://vr.justeasy.cn/view/1xf7951u9o9t0637-1795199079.html
https://vr.justeasy.cn/view/h17k3g951399m110-1795199122.html
https://vr.justeasy.cn/view/yb117795199ua153-1795199164.html
https://vr.justeasy.cn/view/81795y1sk9919v45-1795199206.html
https://vr.justeasy.cn/view/s17qyc951992l3q6-1795199248.html
https://vr.justeasy.cn/view/1ao779510em99279-1795199290.html
https://vr.justeasy.cn/view/e15t457951x99321-1795199332.html
https://vr.justeasy.cn/view/gh179z5k1u9k9363-1795199375.html
https://vr.justeasy.cn/view/0y1kd79514r99759-1795199770.html
https://vr.justeasy.cn/view/o11f795119978s01-1795199812.html
https://vr.justeasy.cn/view/1b7a951998zpk4w3-1795199854.html
https://vr.justeasy.cn/view/11795s19c9f8u816-1795199897.html
https://vr.justeasy.cn/view/1i79ke51t99892q8-1795199940.html
https://vr.justeasy.cn/view/1795199c8au29741-1795199982.html
https://vr.justeasy.cn/view/17j952yv0001t7s3-1795200025.html
https://vr.justeasy.cn/view/1r79ar5h20q0b056-1795200068.html
https://vr.justeasy.cn/view/1qpz79ix5u200099-1795200110.html
https://vr.justeasy.cn/view/q1379yb5200h1461-1795200152.html
https://vr.justeasy.cn/view/17u9c52n80gl0183-1795200195.html
https://vr.justeasy.cn/view/1y79w54200p22uw6-1795200238.html
https://vr.justeasy.cn/view/51795jh200bv26m9-1795200280.html
https://vr.justeasy.cn/view/117j953z52000311-1795200323.html
https://vr.justeasy.cn/view/179520fn503135t4-1795200365.html
https://vr.justeasy.cn/view/1795f52f0039c0e6-1795200408.html
https://vr.justeasy.cn/view/1td795vyii200439-1795200450.html
https://vr.justeasy.cn/view/17i9o5qmi2004c81-1795200493.html
https://vr.justeasy.cn/view/179785w2b00fv524-1795200535.html
https://vr.justeasy.cn/view/1k07j9c520tp0566-1795200578.html
https://vr.justeasy.cn/view/17951e821006t0a9-1795200620.html
https://vr.justeasy.cn/view/v179s5j2y0066x52-1795200663.html
https://vr.justeasy.cn/view/j1bo795203gl0694-1795200706.html
https://vr.justeasy.cn/view/103x79l520d07d37-1795200748.html
https://vr.justeasy.cn/view/9179cd52tg007r79-1795200790.html
https://vr.justeasy.cn/view/1h7j9y8at5200822-1795200833.html
https://vr.justeasy.cn/view/17q97q5op2f00864-1795200875.html
https://vr.justeasy.cn/view/17e8v95820039906-1795200918.html
https://vr.justeasy.cn/view/17975up20049fs50-1795200961.html
https://vr.justeasy.cn/view/17es952001f959b2-1795201004.html
https://vr.justeasy.cn/view/1cyt798552010335-1795201046.html
https://vr.justeasy.cn/view/179520l17zl07jh8-1795201089.html
https://vr.justeasy.cn/view/le117z95201128x0-1795201132.html
https://vr.justeasy.cn/view/bd17195201d81b64-1795201176.html
https://vr.justeasy.cn/view/17w952012k5n9708-1795201220.html
https://vr.justeasy.cn/view/17xw95620k125dz1-1795201263.html
https://vr.justeasy.cn/view/o1795206pu1ac295-1795201307.html
https://vr.justeasy.cn/view/41xro7954g201338-1795201349.html
https://vr.justeasy.cn/view/17f9r53042f01380-1795201391.html
https://vr.justeasy.cn/view/n17958eh2y014g23-1795201434.html
https://vr.justeasy.cn/view/17l9p5v2cs014u64-1795201476.html
https://vr.justeasy.cn/view/17955e12f01i5j06-1795201518.html
https://vr.justeasy.cn/view/w8p17bp9520155x0-1795201561.html
https://vr.justeasy.cn/view/1ebe7952v015g9g3-1795201605.html
https://vr.justeasy.cn/view/1r795m2zt016km37-1795201648.html
https://vr.justeasy.cn/view/1795270w1yda6m80-1795201691.html
https://vr.justeasy.cn/view/179c5207s175mv24-1795201735.html
https://vr.justeasy.cn/view/f179d4u520e17f67-1795201779.html
https://vr.justeasy.cn/view/179dym5tia201811-1795201822.html
https://vr.justeasy.cn/view/f17991l52201z854-1795201866.html
https://vr.justeasy.cn/view/17952p0o18qunl98-1795201912.html
https://vr.justeasy.cn/view/uj179qyh52019544-1795201956.html
https://vr.justeasy.cn/view/r179t5o20t1p9847-1795201999.html
https://vr.justeasy.cn/view/1fe5j7s9520k2031-1795202043.html
https://vr.justeasy.cn/view/170952202do08t74-1795202085.html
https://vr.justeasy.cn/view/1t7c95202110fgg6-1795202128.html
https://vr.justeasy.cn/view/tg51d79f52n02159-1795202171.html
https://vr.justeasy.cn/view/u17th9b652022302-1795202213.html
https://vr.justeasy.cn/view/1upw7n952l0237e5-1795202387.html
https://vr.justeasy.cn/view/0179vw520v524g18-1795202429.html
https://vr.justeasy.cn/view/8147r9e5l2029460-1795202472.html
https://vr.justeasy.cn/view/r13f79d52g025e02-1795202514.html
https://vr.justeasy.cn/view/17p935r2v0254io5-1795202556.html
https://vr.justeasy.cn/view/ri17952g0c2598s7-1795202599.html
https://vr.justeasy.cn/view/1p7952bz00w26i30-1795202641.html
https://vr.justeasy.cn/view/1579520y2w96sk72-1795202684.html
https://vr.justeasy.cn/view/179l52m02gs7m915-1795202726.html
https://vr.justeasy.cn/view/i31p79s5z2027257-1795202768.html
https://vr.justeasy.cn/view/1d7d9t952u027g99-1795202810.html
https://vr.justeasy.cn/view/1m7t905202784wq1-1795202853.html
https://vr.justeasy.cn/view/10795x20n42ow884-1795202895.html
https://vr.justeasy.cn/view/179ftu5nwj202926-1795202937.html
https://vr.justeasy.cn/view/1pi79520y8g2k968-1795202980.html
https://vr.justeasy.cn/view/svs1795828n03010-1795203021.html
https://vr.justeasy.cn/view/s91795rw20303x52-1795203064.html
https://vr.justeasy.cn/view/107955203y0ji925-1795203106.html
https://vr.justeasy.cn/view/178s9l52h0314437-1795203148.html
https://vr.justeasy.cn/view/1q79562qt0g31c79-1795203191.html
https://vr.justeasy.cn/view/187gcev952013222-1795203233.html
https://vr.justeasy.cn/view/z43179542mp03264-1795203275.html
https://vr.justeasy.cn/view/1a73952byg03p306-1795203318.html
https://vr.justeasy.cn/view/l517gy952b703349-1795203360.html
https://vr.justeasy.cn/view/179er520sf3r3991-1795203402.html
https://vr.justeasy.cn/view/175952d5034dp303-1795203445.html
https://vr.justeasy.cn/view/107p952627034z76-1795203487.html
https://vr.justeasy.cn/view/198bm794520k3518-1795203529.html
https://vr.justeasy.cn/view/177d9952a0o35e60-1795203571.html
https://vr.justeasy.cn/view/1ky7952dr03260t2-1795203613.html
https://vr.justeasy.cn/view/71h7935z2r034644-1795203655.html
https://vr.justeasy.cn/view/17g195k203b1r686-1795203698.html
https://vr.justeasy.cn/view/179oqt520372go19-1795203740.html
https://vr.justeasy.cn/view/1795z29038bu77a1-1795203783.html
https://vr.justeasy.cn/view/es107y91520s3814-1795203825.html
https://vr.justeasy.cn/view/17w95kb203q4v856-1795203867.html
https://vr.justeasy.cn/view/18789q452038jp98-1795203910.html
https://vr.justeasy.cn/view/17f9520vf39d4re1-1795203953.html
https://vr.justeasy.cn/view/1rod7mw952036983-1795203995.html
https://vr.justeasy.cn/view/1x71i952c0240q26-1795204037.html
https://vr.justeasy.cn/view/1f79520f490f6vd8-1795204079.html
https://vr.justeasy.cn/view/1c7wph9520411np0-1795204122.html
https://vr.justeasy.cn/view/481795204k1fh503-1795204164.html
https://vr.justeasy.cn/view/17952j041zznc194-1795204206.html
https://vr.justeasy.cn/view/75147g9520hn4237-1795204248.html
https://vr.justeasy.cn/view/13r7v9x52c0y4279-1795204291.html
https://vr.justeasy.cn/view/147933po45204321-1795204333.html
https://vr.justeasy.cn/view/176925z27043cw63-1795204374.html
https://vr.justeasy.cn/view/17f95wi204471065-1795204416.html
https://vr.justeasy.cn/view/17mx9f5i20444pj7-1795204459.html
https://vr.justeasy.cn/view/d1j795e204483u29-1795204503.html
https://vr.justeasy.cn/view/y17795204vk46535-1795204546.html
https://vr.justeasy.cn/view/m1795x2t0a49c577-1795204589.html
https://vr.justeasy.cn/view/1799z52z0nv46619-1795204630.html
https://vr.justeasy.cn/view/w1y7e95lhr204661-1795204673.html
https://vr.justeasy.cn/view/p1795p2qm0n470q3-1795204715.html
https://vr.justeasy.cn/view/167952i350j47446-1795204757.html
https://vr.justeasy.cn/view/176d9ku5k204w789-1795204800.html
https://vr.justeasy.cn/view/17u95a21dtz04831-1795204842.html
