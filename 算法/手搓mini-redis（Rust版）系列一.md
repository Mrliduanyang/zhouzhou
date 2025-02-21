# 手搓mini-redis（Rust版）系列一

## 0.前言

距上一篇分享LZW压缩算法已经过去了几个月了，是时候开新坑了。

最近花了点时间入门了一点Rust，在网上寻找学习资源的时候，发现了这个Tokio官方出的mini-redis，这是一个使用Rust实现的Redis服务端和客户端。但只实现了核心的GET、SET、PUBLISH、SUBSCRIBE功能。简单研究了一下，确实比较适合对初期学习到的各种Rust理念和操作进行一个巩固。

本期先分享mini-redis的数据结构部分。

## 1.mini-redis

### mini-redis数据结构

一切从简，按照最简单最基本的方式来实现Redis。先来想一下，怎么存Redis中的键值对？最简单的方式就是用HashMap，所以先定义一个Redis数据状态类型：

```rust
struct State {
    entries: HashMap<String, Bytes>,
}
```

注意到Redis中的key是有过期时间的，所以我们需要修改下entries的值的类型，增加一个过期时间字段：

```rust
struct Entry {
    data: Bytes,
    expires_at: Option<Instant>,
}

struct State {
    entries: HashMap<String, Entry>,
}

```

这样看起来，存储键值对的部分就比较完善了。

接下来解决过期时间的问题，希望能有一种数据结构能直接取到最早过期的key，同时要考虑到多个key在同一时刻过期的问题，所以我们需要用BTreeSet加元组来存储过期时间，如下所示：

```rust
struct State {
    entries: HashMap<String, Entry>,
    expirations: BTreeSet<(Instant, String)>,
}
```

为什么是BTreeSet，首先BTree部分没啥争议，因为我们需要其中的元素有序，这样才能最快找到最早的过期时间。而Set是为了确保同一个过期时间+key的组合只有一个，想一下，存多个相同的过期时间+key组合也没什么意义。

这样一来，Redis数据存储部分的数据结构就设计完成了。

Redis中还有一项比较重要的功能：发布-订阅，我们来实现它。还得用HashMap，key是订阅的数据键，value则是一个消息发送者，当有一个key需要发送广播消息给订阅该key的订阅者时，直接取出发送者调用发送即可。所以我们进一步扩展State的结构：

```rust
struct State {
    entries: HashMap<String, Entry>,
    expirations: BTreeSet<(Instant, String)>,
    pub_sub: HashMap<String, broadcast::Sender<Bytes>>,
}
```

既然是用来存取数据的，我们要保证最起码的数据一致性，同一个数据，同一时刻，不能有多个写者。用最基本的互斥锁来实现就好，读写的时候都需要先获取锁，之后再进行读写，而没有获取到锁的则等待。我们在State结构上包一层，于是有了：

```rust
struct Shared {
    state: Mutex<State>,
}
```

了解Rust的应该清楚，Rust中通过所有权系统来保证内存安全，但所有权同时也限制了数据共享。而我们的Redis不可能一次只服务一个客户端，那要如何实现多线程之间的数据共享呢？也很简单，Rust提供了Arc（原子引用计数），这个结构通过Clone可以为共享的数据创建一个引用指针，同时增加引用计数器。所以我们在Shared结构上再包一层Arc，就有了：

```rust
struct Db {
    shared: Arc<Shared>,
}
```

### 数据结构之上的Redis操作

有了基础的数据结构定义后，我们来实现所需要的Redis能力。首先构造函数得有：

```rust
impl Db {
    pub(crate) fn new() -> Db {
        let shared = Arc::new(Shared {
            state: Mutex::new(State {
                entries: HashMap::new(),
                pub_sub: HashMap::new(),
                expirations: BTreeSet::new(),
            }),
        });

        Db { shared }
    }
}
```

get操作是一个纯读的操作，所以我们获取state的不可变引用后，获取互斥锁，然后通过HashMap的get方法来获取所需的值：

```rust
impl Db {
    pub(crate) fn get(&self, key: &str) -> Option<Bytes> {
        // lock()获取state上的Mutex的锁，unwrap简单粗暴从Result中获取值
        let state = self.shared.state.lock().unwrap();
        // get()是HashMap上的方法
        // 这里的map是Option上的方法，用来对结果进行处理
        // 如果get()返回了对应的值，则执行如果找到了对应的值,则执行|entry| entry.data.clone()
        // 如果没找到,则返回None
        // clone()不能少，不然所有权就转移了
        state.entries.get(key).map(|entry| entry.data.clone())
    }
}
```

set操作就比较复杂了，需要设置键值对，管理过期时间，发现有过期的key后，还需要通知订阅该key的客户端。我们一步一步来实现：

```rust
impl Db {
    pub(crate) fn set(&self, key: String, value: Bytes, expire: Option<Duration>) {
        // 要修改state，所以要获取state的可变引用。因为使用了Arc，所以这里没有发生所有权转移
        let mut state = self.shared.state.lock().unwrap();
        // 标志位，是否需要发送过期通知
        let mut notify = false;

        let expires_at = expire.map(|duration| {
            // 计算过期时间
            let when = Instant::now() + duration;
            notify = state
                // 辅助函数，从保存过期时间的BTreeSet中获取最近一个过期时间
                .next_expiration()
                // 如果设置的过期时间比最近的过期时间要小，说明需要发送过期通知了
                .map(|expiration| expiration > when) 
                .unwrap_or(true);

            when
        });
        // 插入键值对，注意key要做一次clone，不然所有权转移后，下面代码无法继续再使用key了
        // HashMap的insert方法会返回所插入的值
        let prev = state.entries.insert(
            key.clone(),
            Entry {
                // 这里的value即使转移了所有权，也无所谓，因为后面没有再用到
                data: value,
                // 这里的expires_at后面还有用到，按理来讲不该这么写。但expires_at是一个Option<Instant>类型
                // Instant实现了Copy trait。这意味着当它被使用时，它会被复制而不是移动
                expires_at, 
            },
        );

        if let Some(prev) = prev {
            if let Some(when) = prev.expires_at {
                // 有值，并且有过期时间的话，直接移除过期时间+key的组合
                state.expirations.remove(&(when, key.clone()));
            }
        }

        if let Some(when) = expires_at {
            // 直接添加新的过期时间+key的组合，When是Instant类型
            state.expirations.insert((when, key));
        }
        // 显式调用drop，释放state上的互斥锁
        // 看上去好像不是很有必要显式调用，毕竟在作用域结束时会自动释放锁
        // 如果不提前释放锁，可能会导致以下问题
        // 当前线程持有锁的同时通知后台任务
        // 后台任务被唤醒，立即尝试获取同一个锁
        // 由于当前线程仍持有锁，后台任务被阻塞
        // 可能导致性能问题或在某些情况下引发死锁
        drop(state);

        if notify {
            // 通知后台任务清理过期key，background_task的数据结构还没定义，这里就先假定它可以满足我们的要求
            self.shared.background_task.notify_one();
        }
    }
}
```

publish和subscribe就好办了，操作Tokio提供的多生产者、多消费者通道broadcast即可。

```rust
impl Db {
    pub(crate) fn subscribe(&self, key: String) -> broadcast::Receiver<Bytes> {
        use std::collections::hash_map::Entry;
        
        // 老样子，获取state的可变引用，并获取锁
        let mut state = self.shared.state.lock().unwrap();

        match state.pub_sub.entry(key) {
            // 如果key存在，使用get()获取与key关联的broadcast::Sender<Bytes>
            // 并调用subscribe()创建并返回一个新的接收者
            Entry::Occupied(e) => e.get().subscribe(),
            // 如果key不存在，创建一个新的broadcast通道
            // 使用insert(tx)将发送者插入到HashMap中并返回新创建的接受者
            Entry::Vacant(e) => {
                let (tx, rx) = broadcast::channel(1024);
                e.insert(tx);
                rx
            }
        }
    }

    pub(crate) fn publish(&self, key: &str, value: Bytes) -> usize {
        // 老样子
        let state = self.shared.state.lock().unwrap();

        // 获取key对应的发送者，并发送消息，需要注意处理返回结果是None的情况
        state
            .pub_sub
            .get(key)
            .map(|tx| tx.send(value).unwrap_or(0))
            .unwrap_or(0)
    }
}
```

### 一些需要用到的辅助方法

在完成基本Redis操作外，还会有一些额外的后台操作要进行，比如要在后台任务中清理过期key，以及上面我们还未实现的next_expiration。

next_expiration定义到State上比较合适，因为需要操作存储过期的时间expirations字段。
```rust
impl State {
    fn next_expiration(&self) -> Option<Instant> {
        // 在expirations上创建一个迭代器，并获取迭代器的第一个元素，使用map来安全处理值为None的情况
        // expiration.0是Rust的元组的语法，可以获取元组中的第一个元素 
        self.expirations
            .iter()
            .next()
            .map(|expiration| expiration.0)
    }
}
```

清理过期key的方法命名为purge_expired_keys，并定义到Shared上，因为要操作state字段：

```rust
impl Shared {
    fn purge_expired_keys(&self) -> Option<Instant> {
        let mut state = self.state.lock().unwrap();
        // 这里要上强度了
        // 如果不加下面这行，Rust的借用检查器会阻止我们同时获取entries和expirations两个字段的可变引用
        // 下面这行代码，创建了一个新的可变引用
        // 在循环中，我们通过这个新的可变引用来访问state的不同字段
        // Rust的借用检查器能够理解这种模式，因为我们是通过同一个可变引用来访问不同的字段
        // 而不是直接从state中分别借用这些字段
        // 算是一种常用技巧了
        let state = &mut *state;

        let now = Instant::now();

        // 这行代码中的&和ref涉及到Rust的模式匹配和借用规则
        // &(引用模式)，&在这里是模式匹配的一部分，用于匹配引用
        // state.expirations.iter().next()返回的是一个Option<&(Instant, String)>类型
        // 使用&可以匹配并解构这个引用，获取其指向的值。
        // ref (引用绑定)，ref关键字用于在模式匹配中创建引用
        // 对于key，使用ref意味着我们想要一个对String的引用，而不是移动或复制String
        // 所以&(when, ref key)模式匹配一个包含两个元素的元组的引用
        // 其中的when被直接复制（因为Instant实现了Copy trait）
        // key被作为引用绑定，避免了对String的移动
        while let Some(&(when, ref key)) = state.expirations.iter().next() {
            if when > now {
                return Some(when);
            }
            // 移除过期key
            state.entries.remove(key);
            state.expirations.remove(&(when, key.clone()));
        }
        None
    }
}
```

后台任务

```rust
async fn purge_expired_tasks(shared: Arc<Shared>) {
    // 整体上理解，后台任务需要一直运行，所以放在一个loop中
    // 如果purge_expired_keys有返回值，表示有下一个将要过期的key
    // 那这时候就等待，等到过期时间到达，或者收到清理通知
    // 如果没有将要过期的key，就等待通知即可
    // tokio::select!宏等待两个事件之一完成
    loop {
        if let Some(when) = shared.purge_expired_keys() {
            tokio::select! {
                _ = time::sleep_until(when) => {}
                _ = shared.background_task.notified() => {}
            }
        } else {
            shared.background_task.notified().await;
        }
    }
}
```

## 2.总结

整个设计实现短小精悍，帮助我们学习了HashMap、BTreeSet等数据结构的使用方法，掌握了如何使用Mutex和Arc实现并发控制，了解到了Rust的一些特性，比如所有权系统、借用检查、模式匹配、异步编程（Tokio）。

不得不感叹一下，Rust的学习曲线，太陡峭了！

最后，祝大家新年快乐！


#### 参考资料

https://github.com/tokio-rs/mini-redis

https://claude.ai/login?returnTo=/?#features 学艺不精，很多代码都是claude帮忙解释的
