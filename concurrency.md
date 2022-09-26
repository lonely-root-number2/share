


慢速系统调用/io/定时/pause等

 io：阻塞/非阻塞 < == >多路

future/channel async/await/promise/go 语法糖
rust的exector/go的调度/js的事件循环  调度的问题：出让CPU 
future组合和promise all。  加锁控制共享和转移所有权


go的sysycall：https://github.com/cch123/golang-notes/blob/master/syscall.md
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
