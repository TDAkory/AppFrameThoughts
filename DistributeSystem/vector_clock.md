# [vector clock](https://en.wikipedia.org/wiki/Vector_clock)

Vector Clock是一种轻量级的数据结构，用于在分布式系统中保持数据一致性。它通过给每个节点分配一个矢量时钟来跟踪每个节点对共享资源的修改，这些时钟可以被用来检测数据的冲突，从而保证分布式系统的数据一致性。

核心思想是：每个节点维护一个逻辑时钟向量，向量的长度等于系统中节点的数量（或参与同步的进程数量）。每个元素代表对应节点的时间戳。当节点发生本地事件或接收消息时，会更新自己的时钟向量。

## 工作原理

- 初始化：系统中有n个节点，每个节点维护一个长度为n的向量，初始值全为0。
- 本地事件：当节点i发生本地事件时，它将自己的时钟分量V[i]加1。
- 发送消息：当节点发送消息时，发送进程将自己的逻辑时钟加1(更新自己的向量时钟分量)，然后会将自己的当前向量时钟附加到消息中。
- 接收消息：当节点j收到消息时：
  - 对于向量中的每个元素k，取max(V_local[k], V_received[k])
  - 将自己的时钟分量V[j]加1

## 因果关系判定：

对于两个事件A和B，如果A的向量时钟在所有分量上都小于或等于B的向量时钟(V_A ≤ V_B)，且至少有一个分量严格小于(V_A[i] < V_B[i])，则A happens-before B (A→B)

如果两个向量时钟既不是全小于等于关系，也不是全大于等于关系，则这两个事件是并发的(A||B)

## 简单的例子

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 定义向量时钟结构体
typedef struct {
    int process_count;  // 进程数量
    int* timestamps;    // 每个进程的时间戳数组
} VectorClock;

// 创建一个新的向量时钟
VectorClock* create_vector_clock(int process_count) {
    VectorClock* clock = (VectorClock*)malloc(sizeof(VectorClock));
    clock->process_count = process_count;
    clock->timestamps = (int*)malloc(process_count * sizeof(int));
    memset(clock->timestamps, 0, process_count * sizeof(int));  // 初始化所有时间戳为0
    return clock;
}

// 销毁向量时钟
void destroy_vector_clock(VectorClock* clock) {
    free(clock->timestamps);
    free(clock);
}

// 更新本地进程的时间戳（发生本地事件时调用）
void local_event(VectorClock* clock, int process_id) {
    if (process_id >= 0 && process_id < clock->process_count) {
        clock->timestamps[process_id]++;
    } else {
        fprintf(stderr, "Invalid process ID: %d\n", process_id);
    }
}

// 比较两个向量时钟
// 返回-1（clock1 < clock2），0（等于），1（大于），-2（无法比较）
int compare_vector_clocks(VectorClock* clock1, VectorClock* clock2) {
    if (clock1->process_count != clock2->process_count) {
        fprintf(stderr, "Incompatible vector clocks\n");
        return -2;
    }

    int less = 0, greater = 0;
    
    for (int i = 0; i < clock1->process_count; ++i) {
        if (clock1->timestamps[i] < clock2->timestamps[i]) {
            less = 1;
        } else if (clock1->timestamps[i] > clock2->timestamps[i]) {
            greater = 1;
        }
    }

    if (less && !greater) return -1;
    if (!less && greater) return 1;
    if (!less && !greater) return 0;
    return -2; // 并发，无法比较
}

// 合并两个向量时钟（取每个元素的最大值）
void merge_vector_clocks(VectorClock* dest, VectorClock* src) {
    if (dest->process_count != src->process_count) {
        fprintf(stderr, "Incompatible vector clocks\n");
        return;
    }

    for (int i = 0; i < dest->process_count; ++i) {
        if (src->timestamps[i] > dest->timestamps[i]) {
            dest->timestamps[i] = src->timestamps[i];
        }
    }
}

// 示例用法
int main() {
    // 创建两个向量时钟（假设系统有3个进程）
    VectorClock* clock1 = create_vector_clock(3);
    VectorClock* clock2 = create_vector_clock(3);

    // 模拟进程0和进程2的事件
    local_event(clock1, 0); // 进程0发生本地事件
    local_event(clock1, 0); // 进程0再次发生事件
    local_event(clock1, 2); // 进程2发生事件
    
    local_event(clock2, 1); // 进程1发生事件
    local_event(clock2, 1); // 进程1再次发生事件

    // 比较两个时钟
    int result = compare_vector_clocks(clock1, clock2);
    printf("Comparison result: %d\n", result);

    // 合并时钟
    merge_vector_clocks(clock1, clock2);
    printf("After merging:\n");
    for (int i = 0; i < 3; ++i) {
        printf("Process %d: %d\n", i, clock1->timestamps[i]);
    }

    // 释放资源
    destroy_vector_clock(clock1);
    destroy_vector_clock(clock2);

    return 0;
}
```