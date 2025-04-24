# Epoch Protection

Epoch Protection 是一种用于并发编程的内存管理技术，特别是在无锁数据结构中广泛使用。它通过引入"epoch"（时期）的概念来安全地延迟内存回收，确保不会出现访问已释放内存的情况。下面我将详细解释其原理，并给出具体示例。

Epoch Protection 机制主要包含以下几个核心组件：

- **原子计数器**：系统维护一个原子计数器，称为当前`epoch`（可理解为一个时代或阶段的编号），任何线都可以对其进行递增操作。每个线程也都有一个本地副本，用于记录该线程所处于的`epoch`阶段。
- **共享Epoch表**：所有线程的本地`epoch`值都存储在一个共享的`Epoch`表中，每个线程对应一个缓存行通过这个表，系统可以跟踪所有线程的`epoch`状态。
- **安全Epoch判断**：当所有线程的本地`epoch`值都大于某个全局的`epoch`值`c`时，就认为`c`是安的。此外，系统还维护一个全局的`E_s`，用于记录最大的安全`epoch`。`E_s`会根据共享`Epoch`表的更新同步更新。
- **Trigger Actions**：这是对基本`Epoch`框架的增强。当一个`epoch`变得安全时，可以执行任何全局作。线程在本地递增`epoch`为`X`时，可以关联一个`Action`（回调函数），当`epoch X`变为安全时，系会触发这个回调函数。这个功能通过`drain list`实现，它是一个由`<epoch, action>`组成的操作对列表通过扫描该列表来执行到期的操作，并使用原子的`compare - and - swap`操作确保每个`action`只执行次

其核心思想是：**当一个对象不再被任何线程访问时，延迟两个epoch周期后再安全回收**。这样确保所有可能持有该对象引用的线程都已离开临界区。

## 工作流程

- **Acquire**：为线程`T`创建一个`epoch`，并将线程`T`的本地`epoch`值`E_T`设置为指定的`E`。
- **Refresh**：将线程的本地`epoch`值更新为全局当前`epoch`值，并更新全局最大安全`epoch``E_s`，同时执行`drain list`中就绪的`Actions`。
- **BumpEpoch**：把全局当前`epoch`值更新为当前值加1，并添加一个`<E, Action>`到`drain list中，以便在`E`这个`epoch`变为安全时执行相应的`Action`。
- **Release**：从全局共享的`Epoch`表中删除线程`T`，表示该线程不再参与当前的`epoch`保护机制。

## 为什么需要两个epoch的延迟？

当系统处于epoch N+2时：

- 所有活跃线程的epoch至少为N+1（因为它们会更新到最新epoch）
- 这意味着没有线程会访问epoch N的对象
- 因此可以安全回收epoch N的垃圾列表

这种设计确保了**内存安全**和**无锁并发**的特性。

## 实际应用示例：无锁栈实现

以下是使用Rust的crossbeam-epoch库实现的无锁栈示例：

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::mem::ManuallyDrop;
use std::ptr;
use std::sync::atomic::Ordering::{Acquire, Relaxed, Release};

#[derive(Debug)]
pub struct TreiberStack<T> {
    head: Atomic<Node<T>>,
}

#[derive(Debug)]
struct Node<T> {
    data: ManuallyDrop<T>,
    next: Atomic<Node<T>>,
}

impl<T> TreiberStack<T> {
    pub fn new() -> TreiberStack<T> {
        TreiberStack {
            head: Atomic::null(),
        }
    }

    pub fn push(&self, t: T) {
        let mut n = Owned::new(Node {
            data: ManuallyDrop::new(t),
            next: Atomic::null(),
        });

        let guard = epoch::pin(); // 进入epoch保护区域

        loop {
            let head = self.head.load(Relaxed, &guard);
            n.next.store(head, Relaxed);

            match self.head.compare_and_set(head, n, Release, &guard) {
                Ok(_) => break,
                Err(e) => n = e.new,
            }
        }
    }

    pub fn pop(&self) -> Option<T> {
        let guard = epoch::pin(); // 进入epoch保护区域
        loop {
            let head = self.head.load(Acquire, &guard);

            match unsafe { head.as_ref() } {
                Some(h) => {
                    let next = h.next.load(Relaxed, &guard);

                    if self.head.compare_and_set(head, next, Relaxed, &guard).is_ok() {
                        unsafe {
                            guard.defer_destroy(head); // 延迟销毁节点
                            return Some(ManuallyDrop::into_inner(ptr::read(&(*h).data)));
                        }
                    }
                }
                None => return None,
            }
        }
    }
}
```

在这个实现中：

1. `epoch::pin()`创建一个Guard，将线程标记为活跃并更新本地epoch
2. `defer_destroy()`将要回收的内存放入垃圾列表
3. 当epoch前进足够时，这些内存会被自动回收

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <functional>
#include <memory>

// Epoch 保护管理器
class EpochManager {
public:
    static constexpr int64_t EPOCH_COUNT = 3; // 通常使用3个epoch足够
    
    EpochManager() : global_epoch_(0) {
        for (auto& count : epoch_counts_) {
            count.store(0, std::memory_order_relaxed);
        }
    }
    
    ~EpochManager() {
        for (int64_t e = 0; e < EPOCH_COUNT; ++e) {
            reclaim_objects(e);
        }
    }
    
    // 进入受保护的临界区
    class CriticalSection {
    public:
        CriticalSection(EpochManager& manager) : manager_(manager) {
            // 获取当前线程的epoch并设置为活跃
            local_epoch_ = manager_.global_epoch_.load(std::memory_order_relaxed);
            manager_.epoch_counts_[local_epoch_ % EPOCH_COUNT].fetch_add(1, std::memory_order_relaxed);
        }
        
        ~CriticalSection() {
            // 退出临界区，减少计数
            manager_.epoch_counts_[local_epoch_ % EPOCH_COUNT].fetch_sub(1, std::memory_order_relaxed);
        }
        
        // 延迟回收对象
        template <typename T>
        void defer_delete(T* ptr) {
            int64_t current_epoch = local_epoch_;
            manager_.deferred_objects_[current_epoch % EPOCH_COUNT].emplace_back(
                [ptr]() { delete ptr; }
            );
            
            // 尝试推进全局epoch并回收旧对象
            manager_.try_advance_epoch();
        }
        
    private:
        EpochManager& manager_;
        int64_t local_epoch_;
    };
    
private:
    std::atomic<int64_t> global_epoch_;
    std::atomic<int64_t> epoch_counts_[EPOCH_COUNT];
    std::vector<std::function<void()>> deferred_objects_[EPOCH_COUNT];
    
    void try_advance_epoch() {
        int64_t current_epoch = global_epoch_.load(std::memory_order_relaxed);
        
        // 检查是否可以推进epoch
        for (int64_t e = current_epoch - (EPOCH_COUNT - 1); e <= current_epoch; ++e) {
            if (epoch_counts_[e % EPOCH_COUNT].load(std::memory_order_relaxed) != 0) {
                return; // 仍有线程在使用旧epoch，不能推进
            }
        }
        
        // 可以安全推进epoch
        global_epoch_.fetch_add(1, std::memory_order_relaxed);
        
        // 回收两个epoch前的对象
        int64_t reclaim_epoch = current_epoch - 2;
        if (reclaim_epoch >= 0) {
            reclaim_objects(reclaim_epoch % EPOCH_COUNT);
        }
    }
    
    void reclaim_objects(int64_t epoch_slot) {
        for (auto& deferred : deferred_objects_[epoch_slot]) {
            deferred(); // 执行回收操作
        }
        deferred_objects_[epoch_slot].clear();
    }
};

// 无锁栈实现
template <typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        
        Node(const T& data) : data(data), next(nullptr) {}
    };
    
    std::atomic<Node*> head_;
    EpochManager epoch_manager_;
    
public:
    ~LockFreeStack() {
        Node* current = head_.load(std::memory_order_relaxed);
        while (current) {
            Node* next = current->next;
            delete current;
            current = next;
        }
    }
    
    void push(const T& data) {
        Node* new_node = new Node(data);
        typename EpochManager::CriticalSection guard(epoch_manager_);
        
        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(new_node->next, new_node,
                                          std::memory_order_release,
                                          std::memory_order_relaxed)) {
            // CAS失败，重试
        }
    }
    
    bool pop(T& result) {
        typename EpochManager::CriticalSection guard(epoch_manager_);
        
        Node* old_head = head_.load(std::memory_order_acquire);
        if (!old_head) return false;
        
        while (!head_.compare_exchange_weak(old_head, old_head->next,
                                          std::memory_order_release,
                                          std::memory_order_relaxed)) {
            if (!old_head) return false;
        }
        
        result = old_head->data;
        guard.defer_delete(old_head); // 安全延迟删除
        return true;
    }
};

// 测试代码
int main() {
    LockFreeStack<int> stack;
    const int THREAD_COUNT = 4;
    const int OPERATIONS_PER_THREAD = 1000;
    
    std::vector<std::thread> threads;
    
    for (int i = 0; i < THREAD_COUNT; ++i) {
        threads.emplace_back([&stack, i]() {
            for (int j = 0; j < OPERATIONS_PER_THREAD; ++j) {
                stack.push(i * OPERATIONS_PER_THREAD + j);
                int val;
                if (stack.pop(val)) {
                    // 成功弹出值
                    // std::cout << "Popped: " << val << std::endl;
                }
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << "All operations completed successfully!" << std::endl;
    
    return 0;
}
```

## Epoch Protection 在其他领域的应用

### 1. 区块链中的Epoch概念

在以太坊2.0中，Epoch被用作时间单位，每个Epoch包含32个Slot（每个Slot 12秒），共384秒（约6.4分钟）。验证者委员会在每个Epoch重新随机分配，这种设计类似于Epoch Protection中的epoch概念，确保系统状态的同步和安全更新。

### 2. 数据保护领域

在数据保护系统中，Epoch Data Protection策略允许维护高价值目标文件夹的近线存档，用户可以查看和访问过去选定时间点上的数据存档，类似于"时光机"功能。这在应对勒索软件攻击或意外数据丢失时特别有用。

## Epoch Protection 的优势

1. **无锁并发**：不需要传统的锁机制，提高并发性能
2. **内存安全**：确保不会访问已释放的内存
3. **低延迟**：内存回收操作不会阻塞关键路径
4. **可扩展性**：适用于多核/多线程环境

## 总结

Epoch Protection是一种高效的无锁内存管理技术，通过引入epoch概念和延迟回收机制，在保证内存安全的同时实现高性能并发。它在无锁数据结构、区块链共识机制和数据保护系统等领域都有广泛应用。理解Epoch Protection的原理对于开发高性能并发系统至关重要。