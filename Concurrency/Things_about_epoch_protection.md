# Epoch Protection

Epoch Protection 是一种用于并发编程的内存管理技术，特别是在无锁数据结构中广泛使用。它通过引入"epoch"（时期）的概念来安全地延迟内存回收，确保不会出现访问已释放内存的情况。下面我将详细解释其原理，并给出具体示例。

Epoch Protection 机制主要包含以下几个核心组件：

- **原子计数器**：系统维护一个原子计数器，称为当前`epoch`（可理解为一个时代或阶段的编号），任何线都可以对其进行递增操作。每个线程也都有一个本地副本，用于记录该线程所处于的`epoch`阶段。
- **共享Epoch表**：所有线程的本地`epoch`值都存储在一个共享的`Epoch`表中，每个线程对应一个缓存行通过这个表，系统可以跟踪所有线程的`epoch`状态。
- **安全Epoch判断**：当所有线程的本地`epoch`值都大于某个全局的`epoch`值`c`时，就认为`c`是安的。此外，系统还维护一个全局的`E_s`，用于记录最大的安全`epoch`。`E_s`会根据共享`Epoch`表的更新同步更新。
- **Trigger Actions**：这是对基本`Epoch`框架的增强。当一个`epoch`变得安全时，可以执行任何全局作。线程在本地递增`epoch`为`X`时，可以关联一个`Action`（回调函数），当`epoch X`变为安全时，系会触发这个回调函数。这个功能通过`drain list`实现，它是一个由`<epoch, action>`组成的操作对列表通过扫描该列表来执行到期的操作，并使用原子的`CAS`操作确保每个`action`只执行次

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

下面是[基于Epoch Protection用C++实现的一个无锁Stack](https://godbolt.org/z/oWo1WMbsP)：

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <functional>
#include <memory>
#include <map>

class EpochManager {
public:
    static constexpr int64_t kEpochCount = 3;

    EpochManager() : global_epoch_(0) {
        for (auto& count : epoch_counts_) {
            count.store(0, std::memory_order_relaxed);
        }
        for (int64_t i = 0; i < kEpochCount; ++i) {
            deferred_objects_.push_back(std::map<void *, std::function<void()>>());
        }
    }

    ~EpochManager() {
        for (int64_t e = 0; e < kEpochCount; ++e) {
            ReclaimObj(e);
        }
    }

    class CriticalSection {
        public:
        CriticalSection(EpochManager* epoch_manager) : mgr_(epoch_manager) {
            local_epoch_ = mgr_->global_epoch_.load(std::memory_order_relaxed);
            mgr_->epoch_counts_[local_epoch_ % kEpochCount].fetch_add(1, std::memory_order_relaxed);
        }

        ~CriticalSection() {
            mgr_->epoch_counts_[local_epoch_ % kEpochCount].fetch_sub(1, std::memory_order_relaxed);
        }

        template <typename T>
        void DeferDelete(T *ptr) {
            auto current_epoch = local_epoch_;
            mgr_->deferred_objects_[current_epoch % kEpochCount][(void *)ptr] = [ptr](){delete ptr;};
            mgr_->TryAdvanceEpoch();
        }

    private:
        EpochManager* mgr_;
        int64_t local_epoch_;
    };

private:
    void TryAdvanceEpoch() {
        auto current_epoch = global_epoch_.load(std::memory_order_relaxed);
        for (int64_t e = current_epoch - (kEpochCount - 1); e <= current_epoch; e++) {
            if (epoch_counts_[e % kEpochCount].load(std::memory_order_relaxed) != 0) {
                return;
            }
        }
        global_epoch_.fetch_add(1, std::memory_order_relaxed);
        int64_t reclaim_epoch = current_epoch  - 2;
        if (reclaim_epoch >= 0) {
            ReclaimObj(reclaim_epoch % kEpochCount);
        }
    }

    void ReclaimObj(int64_t reclaim_epoch) {
        for (auto &deferred : deferred_objects_[reclaim_epoch]) {
            deferred.second();
        }
        deferred_objects_[reclaim_epoch].clear();
    }

private:
    std::atomic<int64_t> global_epoch_;
    std::atomic<int64_t> epoch_counts_[kEpochCount];
    std::vector<std::map<void *, std::function<void()>>> deferred_objects_;

};

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
        typename EpochManager::CriticalSection guard(&epoch_manager_);

        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(new_node->next, new_node,
                                          std::memory_order_release,
                                          std::memory_order_relaxed)) {
            // CAS失败，重试
                                          }
    }

    bool pop(T& result) {
        typename EpochManager::CriticalSection guard(&epoch_manager_);

        Node* old_head = head_.load(std::memory_order_acquire);
        if (!old_head) return false;

        while (!head_.compare_exchange_weak(old_head, old_head->next,
                                          std::memory_order_release,
                                          std::memory_order_relaxed)) {
            if (!old_head) return false;
                                          }

        result = old_head->data;
        guard.DeferDelete(old_head); // 安全延迟删除
        return true;
    }
};

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
                    std::cout << "Popped: " << val << std::endl;
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

## Another Example

微软在FasterKV中，基于`Epoch Protection`思想，实现了一个管理并发操作的版本状态机。

首先介绍其`LightEpoch`的实现：

```c#
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT license.

using System;
using System.Threading;
using System.Runtime.InteropServices;
using System.Runtime.CompilerServices;
using System.Diagnostics;

namespace FASTER.core
{
    // Epoch protection
    public unsafe sealed class LightEpoch
    {
        // 与线程相关的静态变量; see https://github.com/microsoft/FASTER/pull/746
        // 这些变量使用 [ThreadStatic] 属性标记，确保每个线程都有自己的副本，在这里我们省略了这些标记
        private class Metadata
        {
            // 线程的唯一标识符
            internal static int threadId;

            // 线程在 epoch 表中的起始偏移量，用于减少探测冲突 (to reduce probing if <see cref="startOffset1"/> slot is already filled)
            // startOffset2 用于在 startOffset1 对应的槽已被占用时提供备用偏移量
            internal static ushort startOffset1;
            internal static ushort startOffset2;

            // 线程在 epoch 表中的条目索引
            internal static int threadEntryIndex;

            // 使用该条目的实例数量
            internal static int threadEntryIndexCount;
        }

        // Size of cache line in bytes
        const int kCacheLineBytes = 64;
        // Default invalid index entry.
        const int kInvalidIndex = 0;
        // Epoch 表的默认大小，取 128 和当前系统处理器核心数两倍中的较大值，以适应不同的并发场景
        static readonly ushort kTableSize = Math.Max((ushort)128, (ushort)(Environment.ProcessorCount * 2));
        // 定义 drain 列表的大小为 16，该列表用于存储需要在某个 Epoch 变为安全可回收时执行的操作
        const int kDrainListSize = 16;

        // 表示 epoch 表中的一个条目
        [StructLayout(LayoutKind.Explicit, Size = kCacheLineBytes)]
        struct Entry
        {
            /// Thread-local value of epoch
            [FieldOffset(0)]
            public long localCurrentEpoch;

            /// ID of thread associated with this entry.
            [FieldOffset(8)]
            public int threadId;

            [FieldOffset(12)]
            public int reentrant;

            [FieldOffset(16)]
            public fixed long markers[6];
        }

        // 用于存储与 epoch 相关联的动作，以便在 epoch 变为安全时执行
        struct EpochActionPair
        {
            public long epoch;
            public Action action;
        }

        // tableRaw是原始的 Epoch 表数组，tableAligned是经过缓存行对齐后的指针，用于更高效的内存访问
        readonly Entry[] tableRaw;
        readonly Entry* tableAligned;

#if !NET5_0_OR_GREATER
        GCHandle tableHandle;
#endif
        // threadIndex是用于存储线程索引的数组，threadIndexAligned是对齐后的指针
        static readonly Entry[] threadIndex;
        static readonly Entry* threadIndexAligned;
#if !NET5_0_OR_GREATER
        static GCHandle threadIndexHandle;
#endif

        // 存储EpochActionPair结构体的数组，每个结构体包含一个 Epoch 值和一个对应的操作
        volatile int drainCount = 0;
        readonly EpochActionPair[] drainList = new EpochActionPair[kDrainListSize];

        // 全局当前的 Epoch 值
        long CurrentEpoch;

        // 缓存当前安全可回收的 Epoch 值
        long SafeToReclaimEpoch;

        // Static constructor to setup shared cache-aligned space to store per-entry count of instances using that entry
        static LightEpoch() { …… }

        // Instantiate the epoch table
        public LightEpoch()
        {
            long p;

            // 在.NET 5.0及以上版本，使用GC.AllocateArray分配数组并获取指针；在旧版本中，通过GCHandle.Alloc进行内存分配和指针获取
#if NET5_0_OR_GREATER
            tableRaw = GC.AllocateArray<Entry>(kTableSize + 2, true);
            p = (long)Unsafe.AsPointer(ref tableRaw[0]);
#else
            // Over-allocate to do cache-line alignment
            tableRaw = new Entry[kTableSize + 2];
            tableHandle = GCHandle.Alloc(tableRaw, GCHandleType.Pinned);
            p = (long)tableHandle.AddrOfPinnedObject();
#endif
            // Force the pointer to align to 64-byte boundaries
            long p2 = (p + (kCacheLineBytes - 1)) & ~(kCacheLineBytes - 1);
            tableAligned = (Entry*)p2;

            CurrentEpoch = 1;
            SafeToReclaimEpoch = 0;

            // Mark all epoch table entries as "available"
            for (int i = 0; i < kDrainListSize; i++)
                drainList[i].epoch = long.MaxValue;
            drainCount = 0;
        }

        // 用于清理 Epoch 表资源
        public void Dispose() { …… }

        // 检查当前线程是否已进入受保护的代码区域，通过判断线程在 Epoch 表中的条目状态来确定
        public bool ThisInstanceProtected()
        {
            int entry = Metadata.threadEntryIndex;
            if (kInvalidIndex != entry)
            {
                if ((*(tableAligned + entry)).threadId == entry)
                    return true;
            }
            return false;
        }

        /// <summary>
        /// Enter the thread into the protected code region
        /// </summary>
        /// <returns>Current epoch</returns>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void ProtectAndDrain()
        {
            int entry = Metadata.threadEntryIndex;

            // Protect CurrentEpoch by making an entry for it in the non-static epoch table so ComputeNewSafeToReclaimEpoch() will see it.
            (*(tableAligned + entry)).threadId = Metadata.threadEntryIndex;
            (*(tableAligned + entry)).localCurrentEpoch = CurrentEpoch;

            if (drainCount > 0)
            {
                Drain((*(tableAligned + entry)).localCurrentEpoch);
            }
        }

        /// <summary>
        /// Thread suspends its epoch entry
        /// </summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void Suspend()
        {
            Release();
            if (drainCount > 0) SuspendDrain();
        }

        /// <summary>
        /// Thread resumes its epoch entry
        /// </summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void Resume()
        {
            Acquire();
            ProtectAndDrain();
        }

        /// <summary>
        /// Increment global current epoch
        /// </summary>
        /// <returns></returns>
        long BumpCurrentEpoch()
        {
            Debug.Assert(this.ThisInstanceProtected(), "BumpCurrentEpoch must be called on a protected thread");
            long nextEpoch = Interlocked.Increment(ref CurrentEpoch);

            if (drainCount > 0)
                Drain(nextEpoch);

            return nextEpoch;
        }

        /// <summary>
        /// Increment current epoch and associate trigger action
        /// with the prior epoch
        /// </summary>
        /// <param name="onDrain">Trigger action</param>
        /// <returns></returns>
        public void BumpCurrentEpoch(Action onDrain)
        {
            long PriorEpoch = BumpCurrentEpoch() - 1;

            int i = 0;
            while (true)
            {
                if (drainList[i].epoch == long.MaxValue)
                {
                    // This was an empty slot. If it still is, assign this action/epoch to the slot.
                    if (Interlocked.CompareExchange(ref drainList[i].epoch, long.MaxValue - 1, long.MaxValue) == long.MaxValue)
                    {
                        drainList[i].action = onDrain;
                        drainList[i].epoch = PriorEpoch;
                        Interlocked.Increment(ref drainCount);
                        break;
                    }
                }
                else
                {
                    var triggerEpoch = drainList[i].epoch;

                    if (triggerEpoch <= SafeToReclaimEpoch)
                    {
                        // This was a slot with an epoch that was safe to reclaim. If it still is, execute its trigger, then assign this action/epoch to the slot.
                        if (Interlocked.CompareExchange(ref drainList[i].epoch, long.MaxValue - 1, triggerEpoch) == triggerEpoch)
                        {
                            var triggerAction = drainList[i].action;
                            drainList[i].action = onDrain;
                            drainList[i].epoch = PriorEpoch;
                            triggerAction();
                            break;
                        }
                    }
                }

                if (++i == kDrainListSize)
                {
                    // We are at the end of the drain list and found no empty or reclaimable slot. ProtectAndDrain, which should clear one or more slots.
                    ProtectAndDrain();
                    i = 0;
                    Thread.Yield();
                }
            }

            // Now ProtectAndDrain, which may execute the action we just added.
            ProtectAndDrain();
        }

        /// <summary>
        /// Mechanism for threads to mark some activity as completed until
        /// some version by this thread
        /// </summary>
        /// <param name="markerIdx">ID of activity</param>
        /// <param name="version">Version</param>
        /// <returns></returns>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void Mark(int markerIdx, long version)
        {
            Debug.Assert(markerIdx < 6);
            (*(tableAligned + Metadata.threadEntryIndex)).markers[markerIdx] = version;
        }

        /// <summary>
        /// Check if all active threads have completed the some
        /// activity until given version.
        /// </summary>
        /// <param name="markerIdx">ID of activity</param>
        /// <param name="version">Version</param>
        /// <returns>Whether complete</returns>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public bool CheckIsComplete(int markerIdx, long version)
        {
            Debug.Assert(markerIdx < 6);

            // check if all threads have reported complete
            for (int index = 1; index <= kTableSize; ++index)
            {
                long entry_epoch = (*(tableAligned + index)).localCurrentEpoch;
                long fc_version = (*(tableAligned + index)).markers[markerIdx];
                if (0 != entry_epoch)
                {
                    if ((fc_version != version) && (entry_epoch < long.MaxValue))
                    {
                        return false;
                    }
                }
            }
            return true;
        }

        /// <summary>
        /// Looks at all threads and return the latest safe epoch
        /// </summary>
        /// <param name="currentEpoch">Current epoch</param>
        /// <returns>Safe epoch</returns>
        long ComputeNewSafeToReclaimEpoch(long currentEpoch)
        {
            long oldestOngoingCall = currentEpoch;

            for (int index = 1; index <= kTableSize; ++index)
            {
                long entry_epoch = (*(tableAligned + index)).localCurrentEpoch;
                if (0 != entry_epoch)
                {
                    if (entry_epoch < oldestOngoingCall)
                    {
                        oldestOngoingCall = entry_epoch;
                    }
                }
            }

            // The latest safe epoch is the one just before the earliest unsafe epoch.
            SafeToReclaimEpoch = oldestOngoingCall - 1;
            return SafeToReclaimEpoch;
        }

        /// <summary>
        /// Take care of pending drains after epoch suspend
        /// </summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        void SuspendDrain()
        {
            while (drainCount > 0)
            {
                // Barrier ensures we see the latest epoch table entries. Ensures
                // that the last suspended thread drains all pending actions.
                Thread.MemoryBarrier();
                for (int index = 1; index <= kTableSize; ++index)
                {
                    long entry_epoch = (*(tableAligned + index)).localCurrentEpoch;
                    if (0 != entry_epoch)
                    {
                        return;
                    }
                }
                Resume();
                Release();
            }
        }

        /// <summary>
        /// Check and invoke trigger actions that are ready
        /// </summary>
        /// <param name="nextEpoch">Next epoch</param>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        void Drain(long nextEpoch)
        {
            ComputeNewSafeToReclaimEpoch(nextEpoch);

            for (int i = 0; i < kDrainListSize; i++)
            {
                var trigger_epoch = drainList[i].epoch;

                if (trigger_epoch <= SafeToReclaimEpoch)
                {
                    if (Interlocked.CompareExchange(ref drainList[i].epoch, long.MaxValue - 1, trigger_epoch) == trigger_epoch)
                    {
                        // Store off the trigger action, then set epoch to int.MaxValue to mark this slot as "available for use".
                        var trigger_action = drainList[i].action;
                        drainList[i].action = null;
                        drainList[i].epoch = long.MaxValue;
                        Interlocked.Decrement(ref drainCount);

                        // Execute the action
                        trigger_action();
                        if (drainCount == 0) break;
                    }
                }
            }
        }

        /// <summary>
        /// Thread acquires its epoch entry
        /// </summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        void Acquire()
        {
            if (Metadata.threadEntryIndex == kInvalidIndex)
                Metadata.threadEntryIndex = ReserveEntryForThread();

            Debug.Assert((*(tableAligned + Metadata.threadEntryIndex)).localCurrentEpoch == 0,
                "Trying to acquire protected epoch. Make sure you do not re-enter FASTER from callbacks or IDevice implementations. If using tasks, use TaskCreationOptions.RunContinuationsAsynchronously.");

            // This corresponds to AnyInstanceProtected(). We do not mark "ThisInstanceProtected" until ProtectAndDrain().
            Metadata.threadEntryIndexCount++;
        }

        /// <summary>
        /// Thread releases its epoch entry
        /// </summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        void Release()
        {
            int entry = Metadata.threadEntryIndex;

            Debug.Assert((*(tableAligned + entry)).localCurrentEpoch != 0,
                "Trying to release unprotected epoch. Make sure you do not re-enter FASTER from callbacks or IDevice implementations. If using tasks, use TaskCreationOptions.RunContinuationsAsynchronously.");

            // Clear "ThisInstanceProtected()" (non-static epoch table)
            (*(tableAligned + entry)).localCurrentEpoch = 0;
            (*(tableAligned + entry)).threadId = 0;

            // Decrement "AnyInstanceProtected()" (static thread table)
            Metadata.threadEntryIndexCount--;
            if (Metadata.threadEntryIndexCount == 0)
            {
                (threadIndexAligned + Metadata.threadEntryIndex)->threadId = 0;
                Metadata.threadEntryIndex = kInvalidIndex;
            }
        }

        // 在 Epoch 表中为线程预留一个条目，通过自旋和尝试获取空闲槽位来实现
        static int ReserveEntry()
        {
            while (true)
            {
                // Try to acquire entry
                if (0 == (threadIndexAligned + Metadata.startOffset1)->threadId)
                {
                    if (0 == Interlocked.CompareExchange(
                        ref (threadIndexAligned + Metadata.startOffset1)->threadId,
                        Metadata.threadId, 0))
                        return Metadata.startOffset1;
                }

                if (Metadata.startOffset2 > 0)
                {
                    // Try alternate entry
                    Metadata.startOffset1 = Metadata.startOffset2;
                    Metadata.startOffset2 = 0;
                }
                else Metadata.startOffset1++; // Probe next sequential entry
                if (Metadata.startOffset1 > kTableSize)
                {
                    Metadata.startOffset1 -= kTableSize;
                    Thread.Yield();
                }
            }
        }

        static int Murmur3(int h) { …… }

        // 该方法为线程分配一个新的 epoch 表条目。它使用线程 ID 的 Murmur3 哈希计算初始偏移量，并通过探查找到可用条目
        static int ReserveEntryForThread()
        {
            if (Metadata.threadId == 0) // run once per thread for performance
            {
                Metadata.threadId = Environment.CurrentManagedThreadId;
                uint code = (uint)Murmur3(Metadata.threadId);
                Metadata.startOffset1 = (ushort)(1 + (code % kTableSize));
                Metadata.startOffset2 = (ushort)(1 + ((code >> 16) % kTableSize));
            }
            return ReserveEntry();
        }
    }
}
```

```c#
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT license.

// modules import

namespace FASTER.core
{
    // 状态机操作的当前状态，显式分配8字节空间（64位）
    // 高8位：Phase（状态阶段标识）
    // 低56位：Version（版本号）
    // REST（0）：稳态标识
    // kIntermediateMask（128）：中间状态标记位
    [StructLayout(LayoutKind.Explicit, Size = 8)]
    public struct VersionSchemeState
    {
        public const byte REST = 0;
        const int kTotalSizeInBytes = 8;
        const int kTotalBits = kTotalSizeInBytes * 8;

        // Phase
        const int kPhaseBits = 8;
        const int kPhaseShiftInWord = kTotalBits - kPhaseBits;
        const long kPhaseMaskInWord = ((1L << kPhaseBits) - 1) << kPhaseShiftInWord;
        const long kPhaseMaskInInteger = (1L << kPhaseBits) - 1;

        // Version
        const int kVersionBits = kPhaseShiftInWord;
        const long kVersionMaskInWord = (1L << kVersionBits) - 1;

        private const byte kIntermediateMask = 128;

        [FieldOffset(0)] internal long Word;

        public byte Phase
        {
            get { return (byte)((Word >> kPhaseShiftInWord) & kPhaseMaskInInteger); }
            set
            {
                Word &= ~kPhaseMaskInWord;
                Word |= (((long)value) & kPhaseMaskInInteger) << kPhaseShiftInWord;
            }
        }

        // whether EPVS is in intermediate state now (transitioning between two states)
        public bool IsIntermediate() => (Phase & kIntermediateMask) != 0;

        public long Version
        {
            get { return Word & kVersionMaskInWord; }
            set
            {
                Word &= ~kVersionMaskInWord;
                Word |= value & kVersionMaskInWord;
            }
        }

        // funciotns
        ……
    }

    // 状态机核心：通过三阶段来转移状态
    // 预计算阶段（GetNextStep）
    // 临界区执行（OnEnteringState）
    // 异步处理阶段（AfterEnteringState）
    public abstract class VersionSchemeStateMachine
    {
        private long toVersion;
        // The actual version this state machine is advancing to, or -1 if not yet determined
        protected internal long actualToVersion = -1;

        ……

        /// <summary> 根据当前状态，计算下一步的状态
        /// Given the current state, compute the next state the version scheme should enter, if any.
        /// </summary>
        /// <param name="currentState"> the current state</param>
        /// <param name="nextState"> the next state, if any</param>
        /// <returns>whether a state transition is possible at this moment</returns>
        public abstract bool GetNextStep(VersionSchemeState currentState, out VersionSchemeState nextState);

        /// <summary> 在进入新状态之前执行的代码（在互斥区域中执行）
        /// Code block to execute before entering a state. Guaranteed to execute in a critical section with mutual
        /// exclusion with other transitions or EPVS-protected code regions 
        /// </summary>
        /// <param name="fromState"> the current state </param>
        /// <param name="toState"> the state transitioning to </param>
        public abstract void OnEnteringState(VersionSchemeState fromState, VersionSchemeState toState);

        /// <summary> 在进入新状态之后执行的代码（可能与其他线程的执行交叉）
        /// Code block to execute after entering the state. Execution here may interleave with other EPVS-protected
        /// code regions. This can be used to collaborative perform heavyweight transition work without blocking
        /// progress of other threads.
        /// </summary>
        /// <param name="state"> the current state </param>
        public abstract void AfterEnteringState(VersionSchemeState state);
    }

    internal class SimpleVersionSchemeStateMachine : VersionSchemeStateMachine
    {
        // Action是FasterKV定义的一些操作回调？？？？
        private Action<long, long> criticalSection;
        ……
    }

    /// Status for state machine execution
    public enum StateMachineExecutionStatus
    {
        OK,
        RETRY,
        FAIL
    }

    // Epoch Protected Version Scheme
    public class EpochProtectedVersionScheme
    {
        private LightEpoch epoch;
        private VersionSchemeState state;
        private VersionSchemeStateMachine currentMachine;

        /// <summary>
        /// Construct a new EPVS backed by the given epoch framework. Multiple EPVS instances can share an underlying
        /// epoch framework (WARNING: re-entrance is not yet supported, so nested protection of these shared instances
        /// likely leads to error)
        /// </summary>
        /// <param name="epoch"> The backing epoch protection framework </param>
        public EpochProtectedVersionScheme(LightEpoch epoch)
        {
            this.epoch = epoch;
            state = VersionSchemeState.Make(VersionSchemeState.REST, 1);
            currentMachine = null;
        }

        /// <summary></summary>
        /// <returns> the current state</returns>
        public VersionSchemeState CurrentState() => state;

        // Atomic transition from expectedState -> nextState
        // 热路径优化，进行激进的内联
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        private bool MakeTransition(VersionSchemeState expectedState, VersionSchemeState nextState)
        {
            if (Interlocked.CompareExchange(ref state.Word, nextState.Word, expectedState.Word) != expectedState.Word) 
                return false;
            Debug.WriteLine("Moved to {0}, {1}", nextState.Phase, nextState.Version);
            return true;
        }

        /// <summary>
        /// Enter protection on the current thread. During protection, no version transition is possible. For the system
        /// to make progress, protection must be later relinquished on the same thread using Leave() or Refresh()
        /// </summary>
        /// <returns> the state of the EPVS as of protection, which extends until the end of protection </returns>
        public VersionSchemeState Enter()
        {
            epoch.Resume();
            TryStepStateMachine();

            VersionSchemeState result;
            while (true)
            {
                result = state;
                if (!result.IsIntermediate()) break;
                epoch.Suspend();
                Thread.Yield();
                epoch.Resume();
            }

            return result;
        }

        /// <summary>
        /// Refreshes protection --- equivalent to dropping and immediately reacquiring protection, but more performant.
        /// </summary>
        /// <returns> the state of the EPVS as of protection, which extends until the end of protection</returns>
        public VersionSchemeState Refresh()
        {
            epoch.ProtectAndDrain();
            VersionSchemeState result = default;
            TryStepStateMachine();

            while (true)
            {
                result = state;
                if (!result.IsIntermediate()) break;
                epoch.Suspend();
                Thread.Yield();
                epoch.Resume();
            }
            return result;
        }

        /// <summary>
        /// Drop protection of the current thread
        /// </summary>
        public void Leave()
        {
            epoch.Suspend();
        }

        internal void TryStepStateMachine(VersionSchemeStateMachine expectedMachine = null)
        {
            var machineLocal = currentMachine;
            var oldState = state;

            // Nothing to step
            if (machineLocal == null) return;

            // Should exit to avoid stepping infinitely (until stack overflow)
            if (expectedMachine != null && machineLocal != expectedMachine) return;

            // Still computing actual to version
            if (machineLocal.actualToVersion == -1) return;

            // Machine finished, but not reset yet. Should reset and avoid starting another cycle
            if (oldState.Phase == VersionSchemeState.REST && oldState.Version == machineLocal.actualToVersion)
            {
                Interlocked.CompareExchange(ref currentMachine, null, machineLocal);
                return;
            }

            // Step is in progress or no step is available
            if (oldState.IsIntermediate() || !machineLocal.GetNextStep(oldState, out var nextState)) return;

            var intermediate = VersionSchemeState.MakeIntermediate(oldState);
            if (!MakeTransition(oldState, intermediate)) return;

            // Avoid upfront memory allocation by going to a function
            StepMachineHeavy(machineLocal, oldState, nextState);
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        private void StepMachineHeavy(VersionSchemeStateMachine machineLocal, VersionSchemeState old, VersionSchemeState next)
        {
            // // Resume epoch to ensure that state machine is able to make progress
            // if this thread is the only active thread. Also, StepMachineHeavy calls BumpCurrentEpoch, which requires a protected thread.
            bool isProtected = epoch.ThisInstanceProtected();
            if (!isProtected)
                epoch.Resume();
            try
            {
                epoch.BumpCurrentEpoch(() =>
                {
                    machineLocal.OnEnteringState(old, next);
                    var success = MakeTransition(VersionSchemeState.MakeIntermediate(old), next);
                    machineLocal.AfterEnteringState(next);
                    Debug.Assert(success);
                    TryStepStateMachine(machineLocal);
                });
            }
            finally
            {
                if (!isProtected)
                    epoch.Suspend();
            }
        }

        /// <summary>
        /// Signals to EPVS that a new step is available in the state machine. This is useful when the state machine
        /// delays a step (e.g., while waiting on IO to complete) and invoked after the step is available, so the
        /// state machine can make progress even without active threads entering and leaving the system. There is no
        /// need to invoke this method if steps are always available. 
        /// </summary>
        public void SignalStepAvailable()
        {
            TryStepStateMachine();
        }

        /// <summary>
        /// Attempt to start executing the given state machine.
        /// </summary>
        /// <param name="stateMachine"> state machine to execute </param>
        /// <returns>
        /// whether the state machine is successfully started (OK),
        /// cannot be started due to an active state machine (RETRY),
        /// or cannot be started because the version has advanced past the target version specified (FAIL)
        /// </returns>
        public StateMachineExecutionStatus TryExecuteStateMachine(VersionSchemeStateMachine stateMachine)
        {
            if (stateMachine.ToVersion() != -1 && stateMachine.ToVersion() <= state.Version) return StateMachineExecutionStatus.FAIL;
            var actualStateMachine = Interlocked.CompareExchange(ref currentMachine, stateMachine, null);
            if (actualStateMachine == null)
            {
                // Compute the actual ToVersion of state machine
                stateMachine.actualToVersion =
                    stateMachine.ToVersion() == -1 ? state.Version + 1 : stateMachine.ToVersion();
                // Trigger one initial step to begin the process
                TryStepStateMachine(stateMachine);
                return StateMachineExecutionStatus.OK;
            }

            // Otherwise, need to check that we are not a duplicate attempt to increment version
            if (stateMachine.ToVersion() != -1 && actualStateMachine.actualToVersion >= stateMachine.ToVersion())
                return StateMachineExecutionStatus.FAIL;

            return StateMachineExecutionStatus.RETRY;
        }


        /// <summary>
        /// Start executing the given state machine
        /// </summary>
        /// <param name="stateMachine"> state machine to start </param>
        /// <param name="spin">whether to spin wait until version transition is complete</param>
        /// <returns> whether the state machine can be executed. If false, EPVS has advanced version past the target version specified </returns>
        public bool ExecuteStateMachine(VersionSchemeStateMachine stateMachine, bool spin = false)
        {
            if (epoch.ThisInstanceProtected())
                throw new InvalidOperationException("unsafe to execute a state machine blockingly when under protection");
            StateMachineExecutionStatus status;
            do
            {
                status = TryExecuteStateMachine(stateMachine);
            } while (status == StateMachineExecutionStatus.RETRY);

            if (status != StateMachineExecutionStatus.OK) return false;

            if (spin)
            {
                while (state.Version != stateMachine.actualToVersion || state.Phase != VersionSchemeState.REST)
                {
                    TryStepStateMachine();
                    Thread.Yield();
                }
            }

            return true;
        }

        /// <summary>
        /// Advance the version with a single critical section to the requested version. 
        /// </summary>
        /// <param name="criticalSection"> critical section to execute, with old version and new (target) version as arguments </param>
        /// <param name="targetVersion">version to transition to, or -1 if unconditionally transitioning to an unspecified next version</param>
        /// <returns>
        /// whether the state machine is successfully started (OK),
        /// cannot be started due to an active state machine (RETRY),
        /// or cannot be started because the version has advanced past the target version specified (FAIL)
        /// </returns>
        public StateMachineExecutionStatus TryAdvanceVersionWithCriticalSection(Action<long, long> criticalSection, long targetVersion = -1)
        {
            return TryExecuteStateMachine(new SimpleVersionSchemeStateMachine(criticalSection, targetVersion));
        }

        /// <summary>
        /// Advance the version with a single critical section to the requested version. 
        /// </summary>
        /// <param name="criticalSection"> critical section to execute, with old version and new (target) version as arguments </param>
        /// <param name="targetVersion">version to transition to, or -1 if unconditionally transitioning to an unspecified next version</param>
        /// <param name="spin">whether to spin wait until version transition is complete</param>
        /// <returns> whether the state machine can be executed. If false, EPVS has advanced version past the target version specified </returns>
        public bool AdvanceVersionWithCriticalSection(Action<long, long> criticalSection, long targetVersion = -1, bool spin = false)
        {
            return ExecuteStateMachine(new SimpleVersionSchemeStateMachine(criticalSection, targetVersion), spin);
        }

    }
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

## Refs

1. Li, Tianyu, Badrish Chandramouli, and Samuel Madden. "Performant Almost-Latch-Free Data Structures Using Epoch Protection." In Data Management on New Hardware (DaMoN’22), June 13, 2022, Philadelphia, PA, USA, 1 - 10. ACM. https://doi.org/10.1145/3533737.3535091.