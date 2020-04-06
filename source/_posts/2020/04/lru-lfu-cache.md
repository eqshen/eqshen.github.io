---
title: 常见的缓存算法及动手实现
date: 2020-04-06 19:28:16
tags:
    - 缓存
    - FIFO
    - LRU
    - LFU
---

# 常见的缓存算法及动手实现

- FIFO 先进先出 - First In First Out.
- LRU 最近最少使用 - Least Recently Used.
- LFU 最不经常使用 - Least Frequently Used.

### FIFO

- 使用队列实现，新增数据从队尾插入，淘汰数据从队头出队。

### LRU

####核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”

####一个操作demo

1. LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

2. cache.put(1, 1);
3. cache.put(2, 2);
4. cache.get(1);       // 返回  1
5. cache.put(3, 3);    // 该操作会使得key=2 作废
6. cache.get(2);       // 返回 -1 (未找到)
7. cache.put(4, 4);    // 该操作会使得key=1 作废
8. cache.get(1);       // 返回 -1 (未找到)
9. cache.get(3);       // 返回  3
10. cache.get(4);       // 返回  4

#### 如何实现这样一个数据结构？一个高效的缓存应该具有以下特点

- 两个方法put(k,v)和get(k)

- 时间复杂度，空间复杂度越低越好，最好是O(1)
- 需要保存每个元素的访问顺序，每次被访问到，就把元素移动到末尾，这里需要链表结构
- 同时还需要hash结构还保证每次查找的O(1)复杂度
- 所以我们需要的是  HashMap + 链表的结合体，在JAVA恰好有LinkedHashMap这种数据结构

#### LinkedHashMap源码分析

在真正动手实现之前我们不妨先看看LinkedHashMap是否满足我们的需求。

先来看看构造函数

```java
public LinkedHashMap() {
    super();
    accessOrder = false;
}
// 这里的 accessOrder 默认是为false，如果要按读取顺序排序需要将其设为 true
// initialCapacity 代表 map 的 容量，loadFactor 代表加载因子 (默认即可)
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

```

accessOrder决定了链表中结点的排序方式

- false :  按照插入顺序排序
- True：按照读取顺序排序

显然，这并不满足我们的需求啊，在LRU中插入/读取都算入“访问”的（从上面的demo就可以看出）

莫方，我们继续看看源码，看看能不能稍加修改为我所用。然后我们就找到了下面三个方法，用于改变结点顺序的。

- **void afterNodeRemoval(Node p) { }**  结点删除后，把对应的数据从链表中删除
- **void afterNodeAccess(Node p) { }** 其作用就是在访问元素之后，将该元素放到双向链表的尾巴处(只有读操作时才会调用）并且在accessOrder = true才生效~~
- **void afterNodeInsertion(boolean evict) { }**  这个貌似是我们的重点啊，我们现在需要的就是把insert也算入“访问”操作，那看看其源码吧

```java
//evict中文意思驱逐，应该是用来控制是否执行删除的一个因素，通过源码追踪，发现只有readObject反序列化的时候才为false,其他时候都为true,所以可以认为这个值始终为true
void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
}
```

然后再来看看`removeEldestEntry`这个方法是有什么用，上源码

```java
     * <p>This method typically does not modify the map in any way,
     * instead allowing the map to modify itself as directed by its
     * return value.  It <i>is</i> permitted for this method to modify
     * the map directly, but if it does so, it <i>must</i> return
     * {@code false} (indicating that the map should not attempt any
     * further modification).  The effects of returning {@code true}
     * after modifying the map from within this method are unspecified.
     *
     * <p>This implementation merely returns {@code false} (so that this
     * map acts like a normal map - the eldest element is never removed).
     *
     * @param    eldest The least recently inserted entry in the map, or if
     *           this is an access-ordered map, the least recently accessed
     *           entry.  This is the entry that will be removed it this
     *           method returns {@code true}.  If the map was empty prior
     *           to the {@code put} or {@code putAll} invocation resulting
     *           in this invocation, this will be the entry that was just
     *           inserted; in other words, if the map contains a single
     *           entry, the eldest entry is also the newest.
     * @return   {@code true} if the eldest entry should be removed
     *           from the map; {@code false} if it should be retained.
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```



函数很简单，但是我特意复制了很长英文注释（保留原汁原味的嘛），意思是这个函数是一个策略函数，用来决定是否移除“最老”的元素，默认是`false`。而我们要实现LRU，当缓存满的时候，必须要移除最老的元素，也就是队头的元素。所以，Override之

```java
//简单易懂
@Override
protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
	return size() > capacity; 
}

```

完整的LRU源码如下：

```java
class LRUCache extends LinkedHashMap<Integer, Integer>{
    //缓存的容量大小
  	private int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }

    public int get(int key) {
      	//如果缓存不存在，返回null或者自定义一个默认值，此处为-1
        return super.getOrDefault(key, -1);
    }

    // put 操作很简单，直接叫爸爸就完事儿了
    public void put(int key, int value) {
        super.put(key, value);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity; 
    }
}

```

就这？没错，就这么简单，因为你站在了巨人的肩膀上~~，当然你也可以从0开始实现。这里就暂时偷懒，插个眼，想进一步了解，看这里 [力扣LRU](https://leetcode-cn.com/problems/lru-cache/solution/)



### LFU

- 核心思想是“如果数据过去被访问数次越多，那么将来被访问的频率也更高”。

#### 如何实现？

- 每个元素需要按照访问次数（频率）排序，当缓存满了，移除访问次数最少的那个；
- 如果缓存中的访问频率都一样，那就移除最近最少使用的（没错，就变成了LRU了）
- get 方法，需要支持O(1)级别的快速查找，考虑了一下果断hash啊
- put方法，最好也支持O(1)级别的，果断用链表啊

#### 举个例子

LFUCache cache = new LFUCache( 2 );  //2为capacity，缓存容量

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回 1
cache.put(3, 3);    // 去除 key 2
cache.get(2);       // 返回 -1 (未找到key 2)
cache.get(3);       // 返回 3
cache.put(4, 4);    // 去除 key 1，此时3和1的访问次数都是2，但1比3来的早。
cache.get(1);       // 返回 -1 (未找到 key 1)
cache.get(3);       // 返回 3
cache.get(4);       // 返回 4

#### 动手实现

采用HashMap和 LinkedHashSet强强联合的方式实现。原理可以看下图(引用自[LeetCode题解)](https://leetcode-cn.com/problems/lfu-cache/solution/ha-xi-biao-shuang-xiang-lian-biao-java-by-liweiwei/)

![](https://pic.leetcode-cn.com/909ea661e76e600e49763d06d2fa72b7897e36ebf47d966292636bce1b241734-image.png)

从上图可以看出，我们需要啥？假设我们缓存的key,value类型都是Integer

- 一个HashMap<Integer,Node> cache;  Node是双向链表的结点类型
- 一个HashMap<Integer,LinkedHashSet<Node>> freqMap；一个记录访问次数的 hash表+双向链表结构，这里就使用JAVA自带的  LinkedHashSet。
- 新来的结点放在双向链表尾部，所以采用`尾插法`，没办法，谁让LinkedHashSet尾插效率高呢~~

首先，定义Node结点结构

```java
class Node {
    int key;
    int value;
  	//默认为1
    int freq = 1;

    public Node() {}
    
    public Node(int key, int value) {
        this.key = key;
        this.value = value;
    }
}

```

然后是，LFU类

```java
public class LFUCache {

    /**
     * 缓存map
     */
    private HashMap<Integer,Node> cache;
    
    private HashMap<Integer,LinkedHashSet<Node>> freqMap;
  
  	/**
     * 避免每次都要遍历freqMap，采用全局变量维护一个当前的最小频次
     */
    private int minFreq;
    
    /**
     * 缓存最大容量
     */
    private int capacity;
    

    public LFUCache(int capacity){
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.freqMap = new HashMap<>();
    }
    
    public void put(int key,int value){
        //TODO
    }
    
    public int get(int key){
        //TODO
        return -1;
    }
}
```

具体的功能需要我们一步一步完善它了。

首先看看 `get`方法，这个应该是最简单的。

- 根据`key`从 `cache`中取出对应的结点Node，
  - 如果Node存在，则返回其值，并node.freq++;
  - 不存在则返回默认值 null ,或者-1(根据业务需求，为了简单这里就 -1吧，实际应该是null,并且结合泛型来做)。

```java
public int get(int key){
        Node node = this.cache.get(key);
        if(node != null){
          //增加此结点的访问次数
            this.freqIncrement(node);
            return node.value;
        }
        return -1;
}
```

增加结点的访问次数不仅仅是  `node.freq++`这么简单，我们要同步更新`this.freqMap`把Node移动到对应频次（调用次数的）桶里。

```java
/**
     * 增加结点访问次数
     * @param node
     */
    public void freqIncrement(Node node){
        //找到旧的桶中的结点，并移除
        int curFreq = node.freq;
        LinkedHashSet<Node> oldSet = this.freqMap.get(curFreq);
        oldSet.remove(node);

        //如果当前桶移除了node后，已经是个空桶了，那么minFreq就要指向大一号的桶
        if(curFreq == minFreq && oldSet.size() == 0){
            minFreq = curFreq + 1;
        }

        //增加访问次数，并加入新桶的链尾
        node.freq++;
        LinkedHashSet<Node> newSet = this.freqMap.get(node.freq);
        if(newSet == null){
            newSet = new LinkedHashSet<>();
            this.freqMap.put(node.freq,newSet);
        }
        newSet.add(node);
    }
```



然后是`put`方法，这个方法就稍复杂一点了。

```java
public void put(int key,int value){
        //容量是0，那它可能是个假缓存吧
        if(this.capacity == 0){
            return;
        }
        Node curNode = cache.get(key);
        //如果缓存已经存在，直接覆盖
        if (curNode != null){
            curNode.value = value;
            this.freqIncrement(curNode);
        }else{
            //缓存容量达到上限了？需要先删除
            if(this.cache.size() == capacity){
                //删除第一个
                LinkedHashSet<Node> set = this.freqMap.get(minFreq);
                Node needRemove = set.iterator().next();
                set.remove(needRemove);
                cache.remove(needRemove);
            }
            //新增
            Node newNode = new Node(key, value);
            cache.put(key,newNode);
            LinkedHashSet<Node> floorDeq = this.freqMap.get(1);
            if(floorDeq == null){
                floorDeq = new LinkedHashSet<>();
                freqMap.put(1,floorDeq);
            }
            floorDeq.add(newNode);
            //更新minFreq
            minFreq = 1;
        }
   }
```

#### 测试

```java
public static void main(String[] args) {
        LFUCache cache = new LFUCache(2);

        cache.put(1, 1);
        cache.put(2, 2);

        System.out.println(cache.getMap().keySet());

        int res1 = cache.get(1);
        System.out.println(res1);

        cache.put(3, 3);
        System.out.println(cache.getMap().keySet());

        int res2 = cache.get(2);
        System.out.println(res2);

        int res3 = cache.get(3);
        System.out.println(res3);

        cache.put(4, 4);
        System.out.println(cache.getMap().keySet());

        int res4 = cache.get(1);
        System.out.println(res4);

        int res5 = cache.get(3);
        System.out.println(res5);

        int res6 = cache.get(4);
        System.out.println(res6);

    }
```





#### 总结

为了代码简单，上面能用java提供的集合类都尽量使用了，但是为了效率高，最好的办法是实现一个自己的双端链表来实现，但是生产环境不推荐自己实现LRU,LFU这种，直接用现成的。

#### 参考资料

- [LeetCode-Sweetiee甜姨](https://leetcode-cn.com/problems/lfu-cache/solution/java-13ms-shuang-100-shuang-xiang-lian-biao-duo-ji/)
- [LeetCode-liwewei大佬](https://leetcode-cn.com/problems/lfu-cache/solution/ha-xi-biao-shuang-xiang-lian-biao-java-by-liweiwei/)