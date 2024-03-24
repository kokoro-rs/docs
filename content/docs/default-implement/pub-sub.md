+++
title = "关于发布订阅者模式"
description = "发布订阅者模式，为什么，是什么，怎么用。"
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
draft = false
weight = 410
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = """发布-订阅模式是一种行为设计模式，它允许多个插件或实例通过事件的发布和订阅来进行通信。
<br/>
在这种模式中，发布者(又称为主题)负责发布事件，而订阅者(也称为观察者)则通过订阅主题来接收这些事件。
<br/>
这种模式使得应用程序的不同部分能够松散耦合，并且可以动态地添加或删除订阅者。"""
toc = true
top = false
+++

## kokoro-flume-channel
既然发布订阅模式很容易实现动态特性，那么 Kokoro 也就没有理由不具备这种能力了。

所以我们有 `kokoro-flume-channel (下称 flume-channel)` 实现以 [flume](https://github.com/zesterer/flume) 为核心的发布-订阅模式。

[flume](https://github.com/zesterer/flume) 的具体特性，请查看其[仓库](https://github.com/zesterer/flume)。

我们对 flume 进行了简单的包装，实现了简单的发布-订阅模式。

### 使用例

```rust
/// 该函数也是 flume-channel 所提供，
/// 用于实例化一个 `Mode` 为 `MPSC` 的 `Context`
/// 其 `Resources` 为 `()`
let ctx = channel_ctx();

/// 注册一个 `Subscriber`
ctx.subscribe(..)
```

这里需要解释，何为 `Subscriber`。

其为 `trait` ，所以实现了 `trait Subscriber` 的任何类型，都可以被注册为订阅者。

默认实现有：

1. `FnMut()`
2. `FnMut(&Context)`
3. `FnMut(&Event)`
4. `FnMut(&Context, &Event)`

实现了 `trait Event` 和 `trait EventID` 的类型可被订阅，`Subscriber` 的 `sub(&Event)` 决定了是否订阅该类型。在 `FuMut` 中，参数类型决定了其订阅类型。
若未给出类型，则默认订阅了 `PhantomEvent`。

```rust
/// 事件 `Hello`
#[derive(Event)]
struct Hello(String);

fn foo(e:& Hello){
    println!("{}", e.0);
}

let ctx = channel_ctx();
ctx.subscribe(foo); /// 订阅了事件 `Hello`
ctx.publish(Hello("Hello World".to_string())); /// 发布了事件 `Hello`
```

此时订阅者并不会被执行，因为发布的原理其实就是 `sender.send`，很明显还需要调用 `receiver.recv`。所以就有：

1. `ctx.run()` 用于迭代 `receiver` (会阻塞线程)
2. `ctx.next()` 用于单次 `recv`
3. `ctx.run_no_block()` 用于非阻塞迭代 `receiver`

2、3 暂未实现

最终我们实现了发布-订阅的 **Hello World**

```rust
#[derive(Event)]
struct Hello(String);
fn foo(e:& Hello){
    println!("{}", e.0);
}
let ctx = channel_ctx();
ctx.subscribe(foo);
ctx.publish(Hello("Hello World".to_string()));
/// 运行
ctx.run()
// 将会输出：Hello World
```

一个事件可以由多个发布者发布，也可以由多个订阅者订阅。需要注意，`kokoro-flume-channel` 是广播而不是单消费者。

**Hello World** 中的发布，并不是常用的发布方式。

发布者应工作在单独的线程中，用于和 `ctx.run()` 协作运行。

至于线程应该什么时候生成，应该什么时候终止。详见 [**关于默认实现 (thread篇)**](#) (还没写)。
