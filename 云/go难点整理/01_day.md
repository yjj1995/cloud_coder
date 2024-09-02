[TOC]
## 数据与切片

### 切片结构



- Slice的底层数据是数组，slice是对数组的封装，它描述一个数组的片段。

  - 数组是定长的，长度定好后，不可改变。
  - 切片可以动态的扩容。

- slice的结构：

  ```
  // runtime/slice.go
  type slice struct {
  	array unsafe.Pointer // 元素指针
  	len   int // 长度 
  	cap   int // 容量
  }
  ```

- 结构示例

  <img src="assets/0.png" alt="切片数据结构" style="zoom:50%;" />

- 底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。



### 切片扩容

- 对于1.18之前的版本来说：
  - 当原slice容量小于`1024`的时候，新slice容量变成原来的`2`倍；
  - 原slice容量超过1024, 新 slice 容量变成原来的`1.25`倍。
- 对于1.18后的版本
  - 当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍；
  - 原slice容量超过256，新slice容量newcap = oldcap+(oldcap+3*256)/4





## benchmark

- 进行性能测试时，尽可能保持测试环境的稳定
- 实现 benchmark 测试
  • 位于 `_test.go` 文件中
  • 函数名以 `Benchmark` 开头
  • 参数为 `b *testing.B`
  • `b.ResetTimer()` 可重置定时器
  • `b.StopTimer()` 暂停计时
  • `b.StartTimer()` 开始计时
- 执行 benchmark 测试
  • `go test -bench .` 执行当前测试
  • `b.N` 决定用例需要执行的次数
  • `-bench` 可传入正则，匹配用例
  • `-cpu` 可改变 CPU 核数
  • `-benchtime` 可指定执行时间或具体次数
  • `-count` 可设置 benchmark 轮数
  • `-benchmem` 可查看内存分配量和分配次数

## Go的defer和return的执行顺序

- 























































