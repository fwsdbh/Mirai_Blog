---
title: 多消费者多生产者无锁队列
published: 2025-10-04
description: ""
image: "./ObjectPool.jpeg"
tags: ["Mirai", "C++", "Concurrent"]
category: C++
draft: false
---



# 多消费者多生产者无锁队列

```c++
#pragma once

#include <atomic>
#include <thread>
#include <vector>
#include <cassert>
#include <cstdint>
#include <functional>
#include <algorithm>
#include <memory>

// -------- Hazard Pointer 简单实现 --------
// 目标：每个线程可以“声明”它正在访问的指针（hazard pointers）。
// 当节点被 retire（延迟删除）时，我们把它放入线程本地的 retire-list。
// 当 retire-list 达到阈值时，扫描全局 hazard pointer 表，
// 对不在任意 hazard pointer 的节点执行 delete。
// 这是最常见的 hazard pointer 回收策略的精简实现。

namespace lf
{

    static const unsigned MAX_HAZARD_POINTERS = 256; // 最大线程数（每个线程一个 hazard slot）,(线程数 * 保护指针数)
    static const unsigned RETIRE_THRESHOLD = 40;     // 每个线程延迟回收阈值

    // 全局 hazard 指针表（简单数组）
    struct HazardRecord
    {
        std::atomic<std::uintptr_t> ptr; // 存放裸指针地址（uintptr_t 便于 atomic）
        HazardRecord() : ptr(0) {}
    };

    static HazardRecord g_hazard_table[MAX_HAZARD_POINTERS];    // 全局 hazard 指针表

    // 为线程分配一个 hazard slot id，（或多个 slot 需要时可扩展）。
    // 这里只分配一个 slot 给每个线程用于保护节点指针（Michael-Scott 需要保护 head/next 等，我们会在需要时临时占用）。
    inline unsigned acquire_hazard_slot_for_current_thread()
    {
        thread_local int slot = -1;
        if (slot != -1)
            return (unsigned)slot;

        for (unsigned i = 0; i < MAX_HAZARD_POINTERS; ++i)
        {
            uintptr_t expected = 0;
            // 试图把 slot 标记为特殊非零值 (例如地址 of &g_hazard_table[i])，表示被占用。
            // 这里我们用 occupant marker: 1 (非 0) 表示已占用。为了简单，先用 CAS 将 0->1 再实际使用 ptr 位置存放 0 表示可用。
            // 实际设计可以用更复杂的注册逻辑。下面采用更保守的操作：先尝试将 ptr 从 0 改为 1 (占位)
            if (g_hazard_table[i].ptr.compare_exchange_strong(expected, 1,
                                                              std::memory_order_acq_rel, std::memory_order_relaxed))
            {
                // 成功占位，初始化为 0（表示当前无保护指针）
                g_hazard_table[i].ptr.store(0, std::memory_order_release);
                slot = (int)i;
                return (unsigned)slot;
            }
        }
        // 如果没有可用 slot，程序利益下会失败。为了安全，抛出或断言。
        // 在生产里应支持更多 slot 或动态注册。
        assert(false && "No hazard pointer slots available: increase MAX_HAZARD_POINTERS");
        return 0;
    }

    // 设置当前线程 slot 的 hazard pointer 值为 p（可以为 nullptr）
    inline void set_hazard_pointer(unsigned slot, void *p)
    {
        assert(slot < MAX_HAZARD_POINTERS);
        g_hazard_table[slot].ptr.store(reinterpret_cast<std::uintptr_t>(p), std::memory_order_release);
    }

    // 清除 slot（设为 nullptr）
    inline void clear_hazard_pointer(unsigned slot)
    {
        assert(slot < MAX_HAZARD_POINTERS);
        g_hazard_table[slot].ptr.store(0, std::memory_order_release);
    }

    // 收集当前所有活跃的 hazard 指针到 vector（用于回收判断）
    inline void collect_hazard_pointers(std::vector<std::uintptr_t> &out)
    {
        out.clear();
        out.reserve(MAX_HAZARD_POINTERS);
        for (unsigned i = 0; i < MAX_HAZARD_POINTERS; ++i)
        {
            uintptr_t p = g_hazard_table[i].ptr.load(std::memory_order_acquire);
            if (p != 0)
                out.push_back(p);
        }
    }

    // 线程本地的 retire list，保存待删除节点指针
    template <typename Node>
    struct RetireList
    {
        std::vector<Node *> nodes;
        RetireList() { nodes.reserve(RETIRE_THRESHOLD * 2); }

        void add(Node *n)
        {
            nodes.push_back(n);
            if (nodes.size() >= RETIRE_THRESHOLD)
            {
                scan_and_reclaim();
            }
        }

        void scan_and_reclaim()
        {
            // 收集所有 hazard 指针
            std::vector<std::uintptr_t> hazards;
            collect_hazard_pointers(hazards);
            // 对于每个待回收节点，若不在 hazards 中则 delete
            auto it = nodes.begin();
            while (it != nodes.end())
            {
                uintptr_t np = reinterpret_cast<std::uintptr_t>(*it);
                bool protected_by_hazard = std::binary_search(hazards.begin(), hazards.end(), np) || (std::find(hazards.begin(), hazards.end(), np) != hazards.end());
                if (!protected_by_hazard)
                {
                    delete *it;
                    it = nodes.erase(it);
                }
                else
                {
                    ++it;
                }
            }
            // 如果仍然很多，可以选择再次尝试或扩大阈值；这里保守退出
        }

        ~RetireList()
        {
            // 程序退出或线程结束时尽量回收残余节点（简单处理）
            for (Node *n : nodes)
                delete n;
            nodes.clear();
        }
    };

} // namespace lf



template <typename T>
class LockFreeQueue
{
private:
    struct Node
    {
        std::atomic<Node *> next;   // 指向下一个节点
        T data;                     // 数据
        bool has_data;              // 是否包含数据（用于判断是否为 dummy 节点）
        Node() : next(nullptr), data(), has_data(false) {}
        explicit Node(const T &v) : next(nullptr), data(v), has_data(true) {}
        explicit Node(T &&v) : next(nullptr), data(std::move(v)), has_data(true) {}
    };

    std::atomic<Node *> head_;  // 队头（指向 dummy 节点）
    std::atomic<Node *> tail_;  // 队尾（指向最后一个节点）

    // 每线程的 retire list
    static lf::RetireList<Node> &thread_retire_list()
    {
        thread_local lf::RetireList<Node> rl;
        return rl;
    }

    // 获取当前线程的 hazard slot id（只分配一次）
    static unsigned get_hazard_slot()
    {
        static thread_local unsigned slot = lf::acquire_hazard_slot_for_current_thread();
        return slot;
    }

public:
    LockFreeQueue()
    {
        Node *dummy = new Node();
        head_.store(dummy, std::memory_order_relaxed);
        tail_.store(dummy, std::memory_order_relaxed);
    }

    ~LockFreeQueue()
    {
        // 删除链上所有节点（此时假设没有并发线程访问）
        Node *p = head_.load(std::memory_order_relaxed);
        while (p)
        {
            Node *n = p->next.load(std::memory_order_relaxed);
            delete p;
            p = n;
        }
    }

    // 禁止拷贝
    LockFreeQueue(const LockFreeQueue &) = delete;
    LockFreeQueue &operator=(const LockFreeQueue &) = delete;

    /**
     * @brief 入队
     * 
     * @param value  要加入队列的元素值
     */
    void Enqueue(const T &value)
    {
        Node *newNode = new Node(value); //  创建新节点
        newNode->next.store(nullptr, std::memory_order_relaxed); //  将新节点的next指针设置为nullptr，使用内存序relaxed

        while (true)
        {
            Node *last = tail_.load(std::memory_order_acquire);         // 获取当前 tail
            Node *next = last->next.load(std::memory_order_acquire);    // 获取 tail 的 next
      
            if (last == tail_.load(std::memory_order_acquire))  // 尝试获取 tail 的独占权
            {
                if (next == nullptr) //  如果 next 为空，说明 tail 是链表的最后一个节点
                {
                    // tail 后没有节点，尝试插入新节点
                    if (last->next.compare_exchange_weak(next, newNode, //  使用比较交换操作尝试将新节点插入到 last 之后
                                                         std::memory_order_release, std::memory_order_relaxed))
                    {
                        // 插入成功，尝试推进 tail 指针到新节点
                        tail_.compare_exchange_strong(last, newNode, //  使用比较交换操作更新 tail 指针
                                                      std::memory_order_release, std::memory_order_relaxed);
                        return;
                    }
                }
                else
                {
                    // tail 后有节点，尝试推进 tail
                    tail_.compare_exchange_strong(last, next,
                                                  std::memory_order_release, std::memory_order_relaxed); //  使用内存顺序为release和relaxed的比较交换操作
                }
            }
            // else retry
        }
    }

    // Enqueue move version
    void Enqueue(T &&value)
    {
        Node *newNode = new Node(std::move(value)); //  创建新节点，使用移动语义初始化节点数据
        newNode->next.store(nullptr, std::memory_order_relaxed); //  将新节点的next指针初始化为nullptr，使用内存序relaxed

        while (true)
        {
            Node *last = tail_.load(std::memory_order_acquire); //  使用memory_order_acquire获取尾节点指针
            Node *next = last->next.load(std::memory_order_acquire); //  获取尾节点的下一个节点指针
            if (last == tail_.load(std::memory_order_acquire)) //  确保尾节点没有被其他线程修改过
            {
                if (next == nullptr)
                {
                    if (last->next.compare_exchange_weak(next, newNode, //  尝试将尾节点的next指针更新为新节点                     使用compare_exchange_weak处理并发情况，成功则使用memory_order_release
                                                         std::memory_order_release, std::memory_order_relaxed))
                    {
                        tail_.compare_exchange_strong(last, newNode, //  更新队列的尾指针为新节点                         使用compare_exchange_strong确保原子性
                                                      std::memory_order_release, std::memory_order_relaxed);
                        return; //  入队成功，退出循环
                    }
                }
                else
                {
                    tail_.compare_exchange_strong(last, next,
                                                  std::memory_order_release, std::memory_order_relaxed);
                }
            }
        }
    }

    // Dequeue (pop). 返回 true 并将 value 设置为弹出值；空队列返回 false
    bool Dequeue(T &value)
    {
        unsigned hp_slot = get_hazard_slot(); //  获取一个危险指针槽位，用于保护正在访问的节点
        while (true)
        {
            Node *first = head_.load(std::memory_order_acquire); //  加载头节点指针，使用memory_order_acquire保证内存顺序
            // 保护 first
            lf::set_hazard_pointer(hp_slot, first); //  设置危险指针，防止节点被其他线程删除
            // 重新读取以保证一致性
            if (head_.load(std::memory_order_acquire) != first) //  检查头节点是否被其他线程修改
            {
                // 被修改，重试
                lf::clear_hazard_pointer(hp_slot); //  清除危险指针，防止内存泄漏
                continue;
            }

            Node *last = tail_.load(std::memory_order_acquire); //  获取尾节点的指针，使用内存获取顺序
            Node *next = first->next.load(std::memory_order_acquire); //  获取首节点的下一个节点指针，使用内存获取顺序

            if (first == head_.load(std::memory_order_acquire)) //  检查当前首节点是否仍然等于head_指针，使用内存获取顺序
            {
                if (next == nullptr) //  检查下一个节点是否为空
                {
                    // 队列空 ，清理并返回
                    lf::clear_hazard_pointer(hp_slot);
                    return false;
                }
                // 保护 next（因为我们要读取 next->data）
                // 为防止同一线程只用一个 slot 导致覆盖，临时我们 reuse same slot:
                // Set hazard to next before CAS head to next (some implementations require two slots).
                // 为简化：先 set next into same slot, 然后 re-check head hasn't changed.
                lf::set_hazard_pointer(hp_slot, next);
                if (head_.load(std::memory_order_acquire) != first)
                {
                    // head changed, retry
                    lf::clear_hazard_pointer(hp_slot);
                    continue;
                }

                // 如果 tail == first（尾落后），尝试推进 tail（帮助）
                if (first == last)
                {
                    tail_.compare_exchange_strong(last, next,
                                                  std::memory_order_release, std::memory_order_relaxed);
                    lf::clear_hazard_pointer(hp_slot);
                    continue;
                }

                // 现在尝试将 head 从 first 推到 next
                if (head_.compare_exchange_strong(first, next,
                                                  std::memory_order_acq_rel, std::memory_order_relaxed))
                {
                    // 成功，取得数据
                    assert(next->has_data); // 除非逻辑出错
                    value = std::move(next->data);

                    // 在把旧节点 retire（延迟删除）前，清除 hazard pointer
                    lf::clear_hazard_pointer(hp_slot);
                    // 把 first 放入 retire list（延迟 delete）
                    thread_retire_list().add(first);
                    return true;
                }
                else
                {
                    // CAS 失败，重试
                    lf::clear_hazard_pointer(hp_slot);
                    continue;
                }
            }
            // else head changed；重试
        }
    }

    /**
     * @brief 检查队列是否为空
     * 
     * @return true 
     * @return false 
     */
    bool Empty() const
    {
        Node *h = head_.load(std::memory_order_acquire);
        Node *n = h->next.load(std::memory_order_acquire);
        return n == nullptr;
    }
};

```
