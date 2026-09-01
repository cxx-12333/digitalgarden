---
{"dg-publish": true, "permalink": "/Blog/go-lease-extender-elegance/", "title": "一段 Go 续期协程的美感——读 multica 的 lease extender", "tags": ["golang", "并发", "源码阅读", "工程哲学"], "created": "2026-08-30", "updated": "2026-08-30", "dg-note-properties": {"title": "一段 Go 续期协程的美感——读 multica 的 lease extender", "permalink": "go-lease-extender-elegance", "created": "2026-08-30", "updated": "2026-08-30", "tags": ["golang", "并发", "源码阅读", "工程哲学"], "type": "blog"}}
---



# 一段 Go 续期协程的美感
<p class="dg-date">✍️ 2026-08-30</p>

读 multica 源码时看到一段任务准备租约的续期代码，四十行不到，却把 Go 并发的几个最佳实践**全部长在了该长的地方**。逐层拆给你看。

```go
func (d *Daemon) startTaskPrepareLeaseExtender(ctx context.Context, task Task, taskLog *slog.Logger) func() {
	leaseCtx, cancel := context.WithCancel(ctx)
	done := make(chan struct{})
	go func() {
		defer close(done)
		ticker := time.NewTicker(taskPrepareLeaseRefresh)
		defer ticker.Stop()
		for {
			select {
			case <-leaseCtx.Done():
				return
			case <-ticker.C:
				reqCtx, reqCancel := context.WithTimeout(leaseCtx, taskPrepareLeaseTimeout)
				err := d.client.ExtendTaskPrepareLease(reqCtx, task.RuntimeID, task.ID)
				reqCancel()
				if err != nil {
					taskLog.Warn("extend task prepare lease failed", "error", err)
				}
			}
		}
	}()

	var once sync.Once
	return func() {
		once.Do(func() {
			cancel()
			<-done
		})
	}
}
```

## 背景：它在解决什么

daemon 领取一个任务后要"准备"（拉镜像、建工作目录），准备期间需要向云端持有一份**租约**（lease）——证明"我还在干活，别把任务派给别人"。租约有 TTL，所以需要一个后台协程周期性续期，直到准备完成。这是个极常见的模式，但常见的写法总在四个地方翻车：**泄漏、双重关闭、卡死等待、以及"请求超时把整个循环带崩"**。这段代码四处全避开了。

## 美感一：返回值是 `func()`，不是通道

API 长这样：`stop := start(...)`，用完 `stop()`。

不用暴露 `done chan` 让调用方去 select，不用暴露 cancel 函数让调用方记着还要等协程退出——**关闭的生命周期被封装成一个动作**。调用方不需要知道里面有协程、有 ticker、有 context，它只知道："调这个，就干净了"。这是 Go 官方博客 [*Go Concurrency Patterns: Context*](https://go.dev/blog/context) 里倡导的形态：把并发细节藏在函数签名后面。

## 美感二：`sync.Once` 兜住双重关闭

```go
var once sync.Once
return func() {
    once.Do(func() {
        cancel()
        <-done
    })
}
```

裸写 `cancel()` + `<-done` 的话：两个 goroutine 同时调 stop，第二次 `<-done` 等一个已关闭的通道倒无害，但 `cancel` 语义上只该发生一次；更危险的是调用方拿这函数到处传。`sync.Once` 把"幂等的停止"做成结构保证，而不是调用方的自觉。**回调可重入，是库代码的基本修养。**

## 美感三：`<-done` 让停止变成同步语义

这是最容易被省略的一行。没有它，`stop()` 返回时协程可能还活着半秒——紧接着的清理代码（删工作目录、释放资源）就和协程里的续期请求**并发跑**，产生幻影般的竞态：日志都打完了，还冒出一条续期记录。

有了 `<-done`：**stop 返回 = 协程已退出，物理保证**。这正对应 7182 那篇 hook engine 的哲学——*"跨越执行上下文时，重新推导每个不变量"*：清理者和后台协程是两个上下文，`<-done` 就是它们之间的 happens-before 边。

## 美感四：三层 context 各司其职

```
ctx（daemon 生命周期）
 └─ leaseCtx（租约续期协程的生命周期，随 stop 取消）
     └─ reqCtx（单次续期请求，taskPrepareLeaseTimeout 超时）
```

关键细节：**单次请求的超时派生自 leaseCtx 而不是裸 context.Background()**。这样 stop 被调用时，即使一个续期请求正卡在网络 IO 上，reqCtx 也会随 leaseCtx 立刻失效——协程不会"卡在最后一次请求"导致 `<-done` 等到超时。三层各自的生命周期：daemon 死 → 一切死；stop 调 → 循环死（含在途请求）；单次请求慢 → 只烧自己的 timeout。

## 美感五：失败是 Warn 不是 Fatal

```go
if err != nil {
    taskLog.Warn("extend task prepare lease failed", "error", err)
}
```

一次续期失败（网络抖动、服务端闪断）只是记警告继续跑——下个 tick 再试。租约天然有 TTL 容忍期，单次失败不会立刻丢任务。对照 7182 的失败分类学：**拒绝/超时/失败要分档治理，一刀切的 panic 或 return 才是错的**。同时注意它没做重试退避——因为 ticker 本身就是节拍器，下一拍就是重试，不引入第二套时间机制。**克制也是设计。**

## 如果要吹毛求疵

两个可议之处（不影响它是一段好代码）：

1. `reqCancel()` 在 err 判断前就调了——位置正确（defer 风格的前置释放），但有人会写成 `defer reqCancel()`，在长循环里 defer 会堆积到函数退出才执行（Go 1.22 前的经典坑）。这里手动的写法反而是更稳的选择。
2. 续期连续失败达到租约 TTL 前没有主动放弃——可以加"连续 N 次失败则 cancel 自己"的快速失败。但这也可能是有意的：让服务端的租约过期机制做最终裁判，客户端不做第二套判断——**一份真相原则**。

## 一图总结

```
stop() ──once──> cancel() ──ctx 级联──> 在途请求也失效
                        └──<-done────> 协程退出确认（happens-before）
ticker ──每 tick──> 带独立超时发续期 ──失败──> Warn，下拍再试
```

> **好的并发代码不是没有协程，而是每个协程都有明确的生老病死，且死亡是可同步确认的。**

这段代码四两拨千斤的地方在于：它没有发明任何东西——Once、WithCancel、WithTimeout、done channel 全是标准件——但每个标准件都装在了最容易错的位置上。所谓功力，就是让平庸的零件长出不平庸的形状。

---

*关联：[我的 PR 败给了什么——一次 webhook 与官方 hook 引擎的哲学差距](Blog/pr-7061-vs-7182-philosophy/) · [文件即接口：agent自动化的调度解耦模式](Notes/文件即接口：agent自动化的调度解耦模式/)*
