### 协程是什么？
+ 更轻量级的线程。在用户态进行调度实现更低成本的任务切换(并发)。  
+ 可以暂停和恢复的函数。（类似于线程池的任务，但又可以暂停恢复，不必等待结束）  
### 有啥用？
优雅的完成一个异步逻辑 
+ 多线程：性能略差
+ 异步回调：丑/回调地狱
+ 协程：平衡了两者的优缺点。线程切换花费的时间成本通常为协程切换的10倍以上。
### 划分
+ 
    + 对称：协程启动了就和启动它的协程无关了，每个都是平等的。 
    + 非对称：返回到调用它的协程。类似于yield
+ 
    + 有栈：类似于线程栈，可从任意地方返回。使用方便
    + 无栈：无线程栈，本质上是个状态机，不能从任意地方返回。性能比有栈协程优秀
### 实现协程的核心：
+ 上下文切换
+ 将阻塞的操作放进用户态的调度器进行管理（意味着什么？从零实现一套标准库将所有相关的syscall等放入管理）【无栈协程主动出让可看作用户自身在控制调度】
+ 调度：抢占式的如何进行抢占/协作式如何主动出让 。（无栈协程通常是协作式，主动出让。有栈协程可抢占）

### 阻塞的操作指的是什么？
慢速系统调用(阻塞的io/pause)，sleep等(比如go中从channel中读写值) 
### 关于几种IO
io：阻塞/非阻塞 < == >多路IO（select/poll/epoll）  
(read/write/send/recv和fcntl)

### 为了实现调度而抽象出来的一些东西
future：表示一个(异步)的操作【对外部来说是可让出CPU】   
channel： 保存异步操作产物的队列（同步）

### 数据竞态的解决(并发模型)

+ 共享内存：加锁控制共享
+ 基于消息（csp/actor）：转移所有权（值类型copy，引用类型copy指针）go中向channel中写入后应不再修改引用指向的数据。rust中，离开代码块时失去所有权，不可修改。
    + csp：通过channel发送消息
    + actor：直接发送消息
```go
func main(){
    // 累加10000次
    // 直接加
	// waitGroup := sync.WaitGroup{}
	// waitGroup.Add(10000)
	x := 0
	for i := 0; i < 10000; i++ {
		go func() {
			x = x + 1
			// waitGroup.Done()
		}()
	}
	// waitGroup.Wait()
	fmt.Println(x) // 不保证为10000

    // 通过channel
    // waitGroup := sync.WaitGroup{}
	// waitGroup.Add(10000)
	x := 0
	ch := make(chan int, 1)
	ch <- x
	for i := 0; i < 10000; i++ {
		go func() {
			tmp_x := <-ch
			tmp_x = tmp_x + 1
			ch <- tmp_x
			// waitGroup.Done()
		}()
	}
	// waitGroup.Wait()
	fmt.Println(<-ch) // 10000
}
```
```rust
fn foo(s:String){}
fn main(){
    let str:String = String::from("hello");
    foo(str);
    println!(str); // 报错
}
```
### 语言的runtime（调度器/执行器）做了哪些工作？
思想:生成任务队列调度执行.
#### js的事件循环
主线程/宏队列/微队列

同步的直接执行
异步的形成任务队列(宏队列/微队列)
#### rust 
runtime将工作分为两部分：  
executor: 执行非阻塞的任务。  
reactor（网络轮询器/多路io的封装）：找出非阻塞的任务交给executor。  

Future:描述任务状态。用户的程序要表达异步逻辑所构造的对象。  
Waker: 几个组件的桥梁.网络轮询器改变Future状态以及将就绪的Future放进executor中. 
==>高性能,能避免无效的轮询
<!-- waker注册到能监视任务状态的的Reactor中如epoll，timer【Waker是由Executor提供】
executor传递waker到future--> 
#### go的调度器
##### GMP模型  
![gmp模型](https://www.liwenzhou.com/images/Go/concurrence/gpm.png)
##### 再来点细节  
如何调用一个系统调用？==>调用规约  
> 按照 linux 的 syscall 调用规范，在汇编中把参数依次传入寄存器，并调用 SYSCALL 指令即可进入内核处理逻辑，系统调用执行完毕之后，返回值放在 RAX 中

| RDI | RSI | RDX | R10 | R8 | R9 | RAX|
|--|--|--|--| --| --|--|
|参数一|参数二|参数三|参数四|参数五|参数六|系统调用编号/返回值|

```go
func Syscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err syscall.Errno)
func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err syscall.Errno)
```
自己实现syscall的目的之一? 调度  
1.14之前的调度实现:
函数和阻塞系统调用插入位点通知runtime调度 
+ 阻塞系统调用：Syscall 插入通知runtime的位点  
+ 非阻塞系统调用：RawSyscall  
+ 不被调度的lowlevel syscall  --->runtime使用  
进入非阻塞的不可被抢占/进入阻塞的系统调用主动让出  
1.14:基于信号的抢占式调度

其它部分:  
channel:将阻塞的go routine放在等待队列，当有其它读写的go routine时挑一个唤醒    
定时器: 调度器内部基于堆实现  
g0:类似与P的职责(创建gouroutine/调度/读写channel/扩容栈)  
sysmon: G是否应该被抢占/网络轮询/GC/抢占调度/检查死锁.... 

### 语法相关
#### 任务生成、组合与同步
+ js:promise/async/await
```js
promise.then()
promise.All(promise1,promise2)
async function foo(){}
await foo()
```  
+ rs:async/await/future组合
```rust
// async:将一个普通函数转换为一个future。【kotlin中称作一个"标明可放弃控制权"的标记】：实质上是改变原来的无法放弃控制权的情况【即一旦阻塞必须等待执行结束  
// Future：描述任务状态。用户的程序要表达异步逻辑所构造的对象。要么是通过基础的 Future经过各种组合子构造，要么通过 async/await构造
```
+ go: channel
#### 不同语言的语法糖   
+ rs：async/await
+ go：go/channel 
+ js：async/await/promise  

#### ==>需要语言编译器层面的支持
### 思考 
为Java实现一套更好用的异步（协程）需要啥？从底层控制这些阻塞调用（即让应用层的这些syscall都通过执行器控制以便出让CPU合理调度），做不到
如果仅依靠线程级别的调度==>性能一般，无意义

#### 趣事 
括号位置的异教徒笑话与unsafexx
actix开源作者因unsafe xx 退出开源

调度的本质：
    改变CPU的使用权（抢占/协作式）

#### kt
job.cancelAndJoin() - 等待协程执行完毕然后再取消  ？ 执行完毕为何还需要取消呢
再看看kt

补充参考：  
阻塞/非阻塞IO与多路复用：https://www.zhihu.com/question/23614342   
go的sysycall：https://github.com/cch123/golang-notes/blob/master/syscall.md  
go设计与实现：https://draveness.me/golang/  
rust异步相关：  
https://zhuanlan.zhihu.com/p/66028983
https://zhuanlan.zhihu.com/p/112237024  
喵老师的rust系列文章：  
https://www.zhihu.com/column/rust-quickstart