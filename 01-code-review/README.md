# justforfunc #1: A Code Review

作者提交的代码修改[MR](https://github.com/philipithomas/iterscraper/pull/1)

说下Review几个要点：

- 简化代码：减少一次性定义变量的使用
- 精简多文件为单文件
- 修复之前goroutine泄露问题：存在主goroutine先退出，子goroutine被动结束的情况
- 有趣的关闭通道实现，如下：

```go
go func() {
    wg.Wait()
    close(results)
}()
```

## 疑问

- 为何使用无缓冲通道（unbuffered chan），多goroutine在无缓冲通道无法发挥并发特性

> 小武 12 小时前
>> Why use tasks/results as unbuffered chan?
> this will block the multi goroutine until read is done?
>
> JustForFunc: Programming in Go 8 小时前
>
>> Why not? Did you prove that was an actual bottleneck before just assuming it is?

所以接下来看下无缓存通道、缓存通道对此是否有影响？
通过对比缓冲队列，得出结论：

- 单一goroutine的tasks写入
- 上述两个均为不耗时操作
- 多goroutine的网络请求，耗时操作在多goroutine中执行
- tasks通道的读取在fetch前，也就是各goroutine不会因为fetch时间较长而阻塞，而是取出数据后tasks则再次写入，因此这部分也不会出现阻塞的情况，真正的瓶颈仍然是fetch操作的网络请求时间
- 单一goroutine的results读取，会不会是瓶颈呢？这个goroutine读取内容写入文件，考虑文件多写异常情况，单一写入更简化了模型且不易出错
- 总结：
    - 分析下来后，这个实现是相对优化，且不受限缓存通道的性能影响，相对均衡的方案
    - 切勿沉迷想当然的结论，代码对比，数据说话

## 附录：无缓冲通道和缓冲通道对比数据

```go
无缓存通道：
time ./unbuffered_iterscraper.exe -from=0 -to=100 -concurrency=5
real    0m4.254s
user    0m0.000s
sys     0m0.031s
```

```go
time ./buffered_iterscraper.exe -from=0 -to=100 -concurrency=5
缓存通道：
real    0m4.238s
user    0m0.000s
sys     0m0.046s
```