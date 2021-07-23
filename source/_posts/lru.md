---
title: LRU算法
date: 2021-07-23 17:15:50
excerpt: LRU算法实现
categories:
- 算法
tags:
- 算法
- 面试
- LRU
---

LRU缓存机制即最少最近使用原则（`Least Recently Used`的缩写），常见于Redis等内存淘汰机制。也是面试的常考题
具体的实现方式是，使用`Map + Node`双向链表实现，Map可以快速定位Node节点，双向链表方便插入和删除。

```java
class LRUCache {

    private int capacity;
    private int size;
    private Node head;
    private Node tail;
    private Map<Integer, Node> map;

    private class Node {
        private Node prev;
        private Node next;
        private int key;
        private int value;
        public Node() {

        }

        public Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    public LRUCache(int capacity) {
        this.head = new Node();
        this.tail = new Node();
        head.next = tail;
        tail.prev = head;
        this.capacity = capacity;
        this.size = 0;
        map = new HashMap<>();
    }
    
    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        Node node = map.get(key);
        removeNode(node);
        moveHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        Node node = map.get(key);
        if (node == null) {
            // 缓存里没有，则需要插入新的节点
            size++;
            if (size > capacity) {
                // 容量已经满了，则需要删除尾节点，并插入新节点
                Node t = tail.prev;
                // 删除node
                map.remove(t.key);
                removeNode(t);
                size--;
            }
            node = new Node(key, value);
            map.put(key, node);
            moveHead(node);
        } else {
            node.value = value;
            removeNode(node);
            moveHead(node);
        }
    }

    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveHead(Node node) {
        // 移动节点到头部
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
        node.prev = head;
    }

}
```
