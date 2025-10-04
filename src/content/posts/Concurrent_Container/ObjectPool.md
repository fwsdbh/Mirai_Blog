---
title: 简单的并发对象池
published: 2025-09-14
description: ""
image: "./ObjectPool.jpeg"
tags: ["Mirai", "C++", "Concurrent"]
category: C++
draft: false
---


```c++
#pragma once

#include <atomic>
#include <vector>
#include <cstddef>
#include <stdexcept>

template <typename T>
class LockFree_ObjectPool
{
private:
    struct Node
    {
        T *object;  // 指向实际对象
        Node *next; // 指向下一个空闲节点
    };

    std::atomic<Node *> head_; // 空闲链表的头
    std::vector<T> storage_;   // 存储实际对象的数组
    std::vector<Node> nodes_;  // 存储空闲节点的数组

public:
    explicit LockFree_ObjectPool(size_t capacity)
        : storage_(capacity), nodes_(capacity)
    {
        if (capacity == 0)
        {
            throw std::invalid_argument("capacity must be greater than 0");
        }

        for (size_t i = 0; i < capacity; ++i)
        {
            nodes_[i].object = &storage_[i];
            nodes_[i].next = (i + 1 < capacity) ? &nodes_[i + 1] : nullptr;
        }
        head_.store(&nodes_[0], std::memory_order_release);
    }

    LockFree_ObjectPool(const LockFree_ObjectPool &) = delete;            // 禁止拷贝构造
    LockFree_ObjectPool &operator=(const LockFree_ObjectPool &) = delete; // 禁止赋值操作

    T *acquire()
    {
        Node *node = pop();
        return node ? node->object : nullptr;
    }

    void release(T *object)
    {
        if (!object)
            return;
        size_t index = static_cast<size_t>(object - &storage_[0]);
        push(&nodes_[index]);
    }

private:
    void push(Node *node)
    {
        Node *oldHead = nullptr;
        do
        {
            oldHead = head_.load(std::memory_order_acquire);
            node->next = oldHead;
        } while (!head_.compare_exchange_weak(oldHead, node, std::memory_order_release, std::memory_order_relaxed));
    }

    Node *pop()
    {
        Node *oldHead = nullptr;

        do
        {
            oldHead = head_.load(std::memory_order_acquire);
            if (!oldHead)
                return nullptr;
        } while (!head_.compare_exchange_weak(oldHead, oldHead->next, std::memory_order_release, std::memory_order_relaxed));

        return oldHead;
    }
};

```