---
title: Redis过期策略
date: {{ date }}
categories:
- 数据源
- Redis
---

**定期删除**：redis每隔100ms就随机抽取一些设置了过期时间的key，检查是否过期，如果过期就删除。

**惰性删除**：在获取某个key的时候，redis会检查一下，这个key如果过期了就会被删除。

**内存淘汰机制**

当redis内存占用过高的时候，此时会进行淘汰，有如下策略

1. noeviction：当内存不足以容纳新写入数据时，新写入被报错
2. allkeys-lru（常用）：当内存不足以容纳新写入数据时，移除最近最久未使用的key
3. allkeys-random：当内存不足以容纳新写入数据时，随机移除某个key
4. volatile-lru：当内存不足以容纳新写入数据时，在设置过期时间的key中，移除最近最最久未使用的key
5. volatile-random：当内存不足以容纳新写入数据时，在设置过期时间的key中，随机移除某个key
6. volatile-ttl：当内存不足以容纳新写入数据时，在设置过期时间的key中，有更早过期时间的key优先移除
7. allkeys-lfu：当内存不足以容纳新写入数据时，移除最近最少使用的key
8. volatile-lfu：当内存不足以容纳新写入数据时，在设置过期时间的key中，移除最近最少使用的key

内存淘汰机制的配置

```powershell
maxmemory-policy noeviction
```

LRU

核心算法：LinkedHashMap

原理：可以重写 `removeEldestEntry` 方法，使得在容量超出 size 的时候，可以执行淘汰策略

```java
public class LruCacheDemo<K, V> extends LinkedHashMap<K, V> {

    private int capacity;   // 缓存容量

    public LruCacheDemo(int capacity){
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return super.size() > capacity;
    }

    public static void main(String[] args) {
        LruCacheDemo<Integer, String> lru = new LruCacheDemo<>(3);
        lru.put(1, "a");
        lru.put(2, "b");
        lru.put(3, "c");
        lru.get(1);
        lru.put(4, "d");
        System.out.println(lru.keySet());
    }
}
```

使用 哈希 + 双向链表 实现

```java
public class LruDemo {

    // 构造一个Node节点作为数据载体
    static class Node<K, V>{
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        public Node(){
            this.prev = this.next = null;
        }

        public Node(K key, V value){
            this.key = key;
            this.value = value;
            this.prev = this.next = null;
        }
    }

    // 构建一个虚拟的双向链表，存放Node
    static class DoubleLinkedList<K, V>{
        Node<K, V> head;
        Node<K, V> tail;

        public DoubleLinkedList(){
            head = new Node<>();
            tail = new Node<>();
            head.next = tail;
            tail.prev = head;
        }

        public void addHead(Node<K, V> node){
            node.next = head.next;
            node.prev = head;
            head.next.prev = node;
            head.next = node;
        }

        public void removeNode(Node<K, V> node){
            node.next.prev = node.prev;
            node.prev.next = node.next;
            node.prev = null;
            node.next = null;
        }

        public Node<K, V> getLast(){
            return tail.prev;
        }
    }

    private int cacheSize;

    Map<Integer, Node<Integer, Integer>> map;

    DoubleLinkedList<Integer, Integer> doubleLinkedList;

    public LruDemo(int cacheSize){
        this.cacheSize = cacheSize;
        map = new HashMap<>();
        doubleLinkedList = new DoubleLinkedList<>();
    }

    public int get(int key){
        if (map.containsKey(key)){
            return -1;
        }
        Node<Integer, Integer> node = map.get(key);
        doubleLinkedList.removeNode(node);
        doubleLinkedList.addHead(node);
        return node.value;
    }

    public void put(int key, int value){
        // 更新
        if (map.containsKey(key)){
            Node<Integer, Integer> node = map.get(key);
            node.value = value;
            map.put(key, node);
            doubleLinkedList.removeNode(node);
            doubleLinkedList.addHead(node);
        } else {
            // 满了 删除链表最后的Node
            if (map.size() == cacheSize){
                Node<Integer, Integer> lastNode = doubleLinkedList.getLast();
                map.remove(lastNode.key);
                doubleLinkedList.removeNode(lastNode);
            }
        }
        // 新增
        Node<Integer, Integer> node = new Node<>(key, value);
        map.put(key, node);
        doubleLinkedList.addHead(node);
    }

    public static void main(String[] args) {
        LruDemo lru = new LruDemo(3);
        lru.put(1, 1);
        lru.put(2, 2);
        lru.put(3, 3);
        lru.get(1);
        lru.put(4, 4);
        System.out.println(lru.map.keySet());
    }
}
```