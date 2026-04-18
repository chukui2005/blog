---
title: 【学习打卡】Day 10 — ArrayList 与 LinkedList 核心知识点
date: 2026-04-15 12:00:00
tags:
  - Java
  - 集合
  - Java后端
categories:
  - 学习打卡
---

# 【学习打卡】Day 10 — ArrayList 与 LinkedList 核心知识点

> 作者：初魁 | 学习周期：Java后端面试冲刺第10天 | 适合人群：Java后端求职者

---

## 一、ArrayList 底层原理

### 1. 底层数据结构

`Object[]` 数组（动态数组）。

### 2. add() 扩容机制

`add(E e)` 流程：

1. 判断 `size + 1` 是否大于当前数组长度
2. 大于 → 触发扩容 `grow()`，新容量 = 旧容量 × 1.5
3. 扩容实际是创建新数组，用 `Arrays.copyOf()` 复制原数组
4. 把元素放入下标 `size` 位置，`size++`

**默认初始容量 10**，懒加载。

### 3. Arrays.asList() 的坑

`Arrays.asList("a", "b", "c")` 返回的不是 `java.util.ArrayList`，而是 `Arrays.ArrayList`——一个**固定长度的内部类**。

- `set()` 修改元素：不会报错，直接修改原数组
- `remove()` 删除元素：**抛 `UnsupportedOperationException`**
- `add()` 添加元素：**抛 `UnsupportedOperationException`**

---

## 二、Vector 与 ArrayList 对比

| | Vector | ArrayList |
|---|---|---|
| 线程安全 | synchronized，线程安全 | 不安全 |
| 扩容 | 2 倍扩容 | 1.5 倍扩容 |
| 性能 | 差（全局锁） | 好 |

实际回答：*"Vector 是早期 JDK 的同步列表，现在基本用 ArrayList；需要线程安全时用 `CopyOnWriteArrayList` 或 `Collections.synchronizedList()` 包装。"*

---

## 三、Iterator 迭代器失效问题

**问题**：Iterator 迭代 ArrayList 时，用 `list.remove()` 边迭代边删除会**抛 `ConcurrentModificationException`**（并发修改异常）。

**原因**：迭代器内部维护了 `expectedModCount = modCount`。用 `list.remove()` 删除元素时 `modCount++`，但迭代器的 `expectedModCount` 没变，检查时发现不一致就抛异常。

**正确做法**：用 `iterator.remove()` 删除。

---

## 四、LinkedList 时间复杂度

| 操作 | 时间复杂度 |
|------|-----------|
| 按 index 查 | O(n) |
| 按值查 | O(n) |
| 头部删 | O(1) |
| 尾部删 | O(1) |
| 头部插 | O(1) |
| 尾部插 | O(1) |

**ArrayList vs LinkedList**：

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| 按 index 查 | O(1) | O(n) |
| 按值查 | O(n) | O(n) |
| 头部删 | O(n) | O(1) |
| 尾部删 | O(1) | O(1) |

---

## 五、数据结构：二叉树

### 二叉树遍历

| 方式 | 顺序 | 应用 |
|------|------|------|
| 前序遍历 | 根 → 左 → 右 | 树的创建/复制 |
| 中序遍历 | 左 → 根 → 右 | 二叉搜索树（BST）排序 |
| 后序遍历 | 左 → 右 → 根 | 树的删除/释放 |

### 二叉搜索树（BST）

**特点**：左子树所有节点 < 根节点 < 右子树所有节点。

**时间复杂度**：O(log n)（树高平衡时）；O(n)（退化成链表时）。

---

> 说明：本节基于黑马课程《07-常见集合篇》P1-9 内容整理。
