---
title: 单消费者单生产者无锁队列
published: 2025-10-04
description: ""
image: "./ObjectPool.jpeg"
tags: ["Mirai", "C++", "Concurrent"]
category: C++
draft: false
---

# 单消费者单生产者无锁队列

```c++
#pragma once
#include <atomic>
#include <vector>
#include <cstddef>
#include <cassert>

template <typename T>
class LockFreeQueueSPSC
{
public:
    explicit LockFreeQueueSPSC(size_t capacity)
        : buffer_(capacity + 1), capacity_(capacity + 1),
          head_(0), tail_(0) {}

    // 拷贝/移动入队
    bool enqueue(const T &item) { return enqueue_impl(item); }
    bool enqueue(T &&item) { return enqueue_impl(std::move(item)); }

    // 原地构造入队
    template <typename... Args>
    bool emplace(Args &&...args)
    {
        size_t tail = tail_.load(std::memory_order_relaxed);
        size_t next_tail = (tail + 1) % capacity_;
        if (next_tail == head_.load(std::memory_order_acquire))
            return false; // 队列满

        buffer_[tail] = T(std::forward<Args>(args)...);
        tail_.store(next_tail, std::memory_order_release);
        return true;
    }

    bool dequeue(T &item)
    {
        size_t head = head_.load(std::memory_order_relaxed);
        if (head == tail_.load(std::memory_order_acquire))
            return false; // 队列空

        item = std::move(buffer_[head]);
        head_.store((head + 1) % capacity_, std::memory_order_release);
        return true;
    }

private:
    template <typename U>
    bool enqueue_impl(U &&item)
    {
        size_t tail = tail_.load(std::memory_order_relaxed);
        size_t next_tail = (tail + 1) % capacity_;
        if (next_tail == head_.load(std::memory_order_acquire))
            return false; // 队列满

        buffer_[tail] = std::forward<U>(item);
        tail_.store(next_tail, std::memory_order_release);
        return true;
    }

private:
    std::vector<T> buffer_;     // 队列缓冲区
    const size_t capacity_;     // 队列容量
    std::atomic<size_t> head_;  // 队头计数器
    std::atomic<size_t> tail_;  // 队尾计数器
};

```
