### 协程是什么？
更轻量级的线程。在用户态进行调度实现更低成本的任务切换(并发)。
### 实现协程的核心：
+ 将阻塞的操作放进用户态的调度器进行管理（意味着什么？从零实现一套标准库将所有相关的syscall等放入管理）
+ 调度：抢占式的如何进行抢占/协作式如何主动出让 

### 阻塞的操作指的是什么？
慢速系统调用(阻塞的io/pause)，sleep等(比如go中从channel中读写值) 
#### 关于几种IO
io：阻塞/非阻塞 < == >多路IO  
(read/write/send/recv和fcntl)

### 为了实现调度而抽象出来的一些东西
future：表示一个(异步)的操作【对外部来说是可让出CPU】 

channel： 保存异步操作产物的队列

### 任务组合与同步
future组合/promise.All和then/channel

不同语言的语法糖  
    rs：async/await/
    go：语法糖 
    js：async/await/promise


语言的runtime（调度器/执行器）做了哪些工作
    rust的exector
    go的调度器
    js的事件循环  

数据竞态的解决
    加锁控制共享和转移所有权

go的sysycall：https://github.com/cch123/golang-notes/blob/master/syscall.md
==>
主动让出：RawSyscall未通知runtime进入syscall以及在函数调用插入位点
抢占式调度：1.14，信号
阻塞系统调用：Syscall
非阻塞系统调用：RawSyscall

不被调度的lowlevel syscall  --->runtime使用
进入非阻塞的不可被抢占
进入阻塞的系统调用主动让出

# Rust
runtime将工作分为两部分：
executor: 执行非阻塞的任务。：tokio、smol、async-std
reactor：找出非阻塞的任务交给executor。

Future：用户的程序要表达异步逻辑所构造的对象。要么是通过基础的 Future经过各种组合子构造，要么通过 async/await构造

async:将一个普通函数转换为一个future。【kotlin中称作一个"标明可放弃控制权"的标记】：实质上是改变原来的无法放弃控制权的情况【即一旦阻塞必须等待执行结束

waker注册到能监视任务状态的的Reactor中如epoll，timer【Waker是由Executor提供

executor传递waker到future


https://zhuanlan.zhihu.com/p/66028983
https://zhuanlan.zhihu.com/p/112237024

https://mmn36.com/tags?page=2

https://nnp35.com/upload_json_live/20210813/videolist_20210813_00_2_-_-_100_4.json

# kt
job.cancelAndJoin() - 等待协程执行完毕然后再取消  ？ 执行完毕为何还需要取消呢
再看看kt

# 思考
为Java实现一套更好用的异步（协程）需要啥？从底层控制这些阻塞调用（即让应用层的这些syscall都通过执行器控制以便出让CPU合理调度），做不到
如果仅依靠线程级别的调度==>性能一般，无意义

# 趣事
括号位置的异教徒笑话与unsafexx
actix开源作者因unsafe xx 退出开源




调度的本质：
    改变CPU的使用权（抢占/协作式）


补充阅读：
阻塞/非阻塞IO与多路复用：https://www.zhihu.com/question/23614342
go的sysycall：https://github.com/cch123/golang-notes/blob/master/syscall.md