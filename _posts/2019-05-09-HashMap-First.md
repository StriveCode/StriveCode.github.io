---
layout: post
title: "HashMap源码分析（上）"
subtitle: "记录一下源码学习过程"
Date: 2019-05-09 10:18:00
author: "Henton"
header-img: "img/post-bg-blog-gitalk.jpg"
tags:
  - JDK
---

> HashMap是一种散列表，通过数组加链表的方式实现。在JDK1.8版本后新增加了红黑树结构来提高效率。

## HashMap

HashMap是Java的Map接口的一种基础哈希表实现。它提供了Map的所有操作，并且支持以null为key或value。（除了是非同步的以及支持null以外，HashMap与HashTable基本相同）HashMap是无序的，即每次遍历的顺序都可能不同。

假如hash算法将每个元素均匀的分散在不同的桶空间（bucket）中，那么HashMap的get和put方法的时间性能将会是常量级别（O(1)）。迭代HashMap所需的时间与其容量（桶空间的数量）和大小（键值对的数量）之和成正比。所以，如果迭代性能比较重要，最好不要将初始容量设置的太大或负载因子设置的过小。

一个HashMap实例的性能将受到两个参数的影响，分别是初始容量（initial capacity）和负载因子（load factor）。容量就是HashMap中桶空间的数量，显然初始容量就是HashMap被初始化时的容量。负载因子则是衡量什么时候HashMap的容量会被自动增加。当HashMap的大小达到容量和负载因子的乘积时，HashMap将会rehash（内部数据结构将会重新构建），rehash之后HashMap的容量大约是之前的两倍。

一般来说，默认的负载因子（0.75）较好的平衡了时间复杂度和空间复杂度。调高负载因子的值可以减少所需的空间大小但是会增加查找耗时（影响到HashMap的许多操作，包括常用的get和put）。在初始化HashMap时应该根据预期要存储的键值对数量和期望的负载因子的值来设置一个合适的初始容量，从而尽量减少rehash的次数。如果设置的初始容量大于键值对可能的最大数量值除以负载因子，那么rehash将永远不会发生。

如果一个HashMap实例要存储大量的键值对，那么设置一个足够大的初始容量会比让它自动rehash扩容存储效率更高。注意，不论是哪一种哈希表，如果大量的key的hash值相同，那么其效率一定会降低。为了减轻这种影响，当key实现了Comparable接口，HashMap会利用key之间的比较顺序来帮助打破联系（红黑树节点时有用）。

一定要注意HashMap不是同步的，即非线程安全。如果多个线程并发的访问同一个HashMap，并且有一个线程及以上的线程在结构上修改这个HashMap，那么一定要在外部保证同步访问。（在结构上修改是指添加或删除一个或多个键值对，而不是仅仅是修改一个已经存在的key对应的value。）一种经典的做法是在封装了Map的对象中进行同步。

如果没有这样的对象，那么应该使用Collections.synchronizedMap()方法来包装Map。最好在创建对象时就这么做，从而避免意料之外的非同步访问。

```java
Map m = Collections.synchronizedMap(new HashMap(...));
```

任何方式获得的HashMap的迭代器在创建后，都会由于除了当前迭代器本身的remove方法之外导致的对应实例发生结构上的改变而发生fail-fast，此时迭代器将抛出ConcurrentModificationException异常。也就是说，当发生并发修改时迭代器会干脆利落的停止执行，而不是留下一个未知的风险。

注意迭代器的fail-fast行为不能保证一定会触发，因为一般而言，无法绝对确认非同步的并发修改的发生。迭代器只能尽可能地抛出ConcurrentModificationException异常。因此依赖这个异常的程序可能会发生错误。迭代器的fail-fast行为应该只被用来发现bug。

## 默认参数

```java
/**  
 * 默认的初始容量，必须是2的幂。  
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**  
 * 最大容量，当试图设置一个大于该值的容量时HashMap会在内部使用该值。设置容量必须是2的幂且小于等于2的30次方。  
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**  
 * 负载因子的默认值。  
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**  
 * 以list形式存储桶节点数量的上限，达到（大于等于）这个上限后再继续添加节点就会转换成红黑树的形式存储。这个值   
 * 必须大于2并且应该大于8来配合红黑树节点移除时转换为普通链表的假设（即要大于UNTREEIFY_THRESHOLD，防止频繁  
 * 的转换）。  
 */
static final int TREEIFY_THRESHOLD = 8;

/**  
 * 以红黑树形式存储桶节点数量的下限，低于这个下限则转换成list的形式存储。这个值应该小于TREEIFY_THRESHOLD，  
 * 并且最大为6来配合移除红黑树节点时的收缩检测（即转换为普通链表）.  
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**  
 * 桶可以被转换为红黑树结构的最小容量，如果HashMap当前容量小于这个值，那么即使一个桶中的节点数量超过  
 * TREEIFY_THRESHOLD也不会被转换为红黑树结构，而是会发生rehash。为了避免，应该至少是TREEIFY_THRESHOLD  
 * 的4倍，  
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 节点

```java
/**  
 * 哈希表的基础桶节点，大部分键值对都是这个类型。（HashMap中的TreeNode和LinkedHashMap中的Entry都是它的  
 * 子类）  
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

## 内部静态方法

```java
/**  
 * 计算key.hashCode()并将结果的高16位与低16位进行异或。因为桶数组（table）的大小为2的幂，所以只有部分bit   
 * 位是有效的。（如当前容量为16，那么只有后4位有效）众所周知的例子是所有的float数都可以作为key存储在一个小容   
 * 量的HashMap中。所以将高位的影响向下传递。这是一种折衷了速度、实用性和质量的方法。因为许多常见的hash值已经   
 * 很好的分散了（所以不会从该方法获益），并且当桶中冲突严重时会转换为红黑树来处理，所以采用了高位地位相异或这  
 * 种性能消耗很低的方法来尽可能的避免系统性的缺陷，也是为了让高位也能产生作用，否则由于HashMap容量的限制，高   
 * 位数将永远不会被用到。  
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**  
 * 如果x类定义具有"class C implements Comparable<C>"的形式，那么返回x的Class，否则返回null  
 */
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (Type t : ts) {
                if ((t instanceof ParameterizedType) &&
                    ((p = (ParameterizedType) t).getRawType() ==
                     Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}

/**  
 * 如果x不等于null且是kc类的实例，返回k.compareTo(x)的结构，否则返回0。  
 */
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}

/**  
 * 根据给定的cap返回一个2的幂数。  
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## 属性

```java
/**  
 * 桶数组，首次使用时进行初始化并且会在必要时进行rehash来扩容。当对其初始化时，length永远是2的幂。（在某些   
 * 操作中，length可以为0便于实现允许懒加载。）  
 */
transient Node<K,V>[] table;

/**  
 * 缓存entrySet()方法的结果，注意keySet()和values()方法的缓存结果使用的是AbstractMap中的属性keySet和  
 * values。  
 */
transient Set<Map.Entry<K,V>> entrySet;

/**  
 * HashMap中键值对的数量。  
 */
transient int size;

/**  
 * 当前HashMap实例被结构化修改的次数。结构化修改是指改变HashMap中键值对的数量或者修改了内部结构（例如  
 * rehash）。这个属性被用来实现迭代器的fail-fast(参考 ConcurrentModificationException)。  
 */
transient int modCount;

/**  
 * 当键值对数量超过这个值之后HashMap就会进行rehash操作（容量 * 负载因子）。注意，当桶数组未被初始化时这个值   
 * 等于初始容量，当未显示指定HashMap的初始容量时这个值为0（代表默认初始容量DEFAULT_INITIAL_CAPACITY）。   
 *  
 * @serial  
 */
int threshold;

/**  
 * 哈希表的负载因子。  
 *  
 * @serial  
 */
final float loadFactor;
```

## 构造方法

```java
/**  
 * 以给定的初始容量和负载因子构造一个空的HashMap。  
 *  
 * @param  initialCapacity 初始容量。  
 * @param  loadFactor      负载因子。  
 * @throws IllegalArgumentException 如果initialCapacity是负数或loadFactor是非正数则会抛出  
 * IllegalArgumentException。  
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

/**  
 * 以给定的初始容量和默认的负载因子（0.75）构造一个空的HashMap。  
 *  
 * @param  initialCapacity 初始容量  
 * @throws IllegalArgumentException 如果initialCapacity是负数或loadFactor是非正数则会抛出  
 * IllegalArgumentException。  
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**  
 * 以默认的初始容量（16）和默认的负载因子（0.75）构造一个空的HashMap。  
 */  
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**  
 * 构造一个与给定Map具有相同的键值对的HashMap。（注：新Map中对应的key和value指向同一个对象）  
 * 新构造的HashMap的负载因子为默认的负载因子（0.75），初始容量则是足够大的值，可以存储给的Map的所有键值对。  
 *  
 * @param   m 要存储的键值对所属的Map。  
 * @throws  NullPointerException 当m为null时抛出该异常。  
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

## 基本方法

### put方法

```java
/**  
 * 将给定的key和value关联起来并存储到HashMap中。如果已经存在给定key的键值对，旧value将被新value替换。  
 *  
 * @param key 给定的value将要与之相关联。  
 * @param value 将要与key相关联。  
 * @return 与key相关联的旧value, 如果没有则返回null。当然返回null也可能是与key相对应的旧value就是  
 * null。  
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

/**  
 * Map.put及相关方法的实现。  
 *  
 * @param hash key的hash值  
 * @param key 要操作的key值  
 * @param value 要操作的value值  
 * @param onlyIfAbsent 如果是true，则不更新旧值（即只能进行插入操作）。  
 * @param evict 如果是false则代表是创建模式。  
 * @return 旧的value值，如果没有则返回null。  
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)//检查当前桶数组是否需要初始化。
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)//根据hash找到key在桶数组中的索引，p为第一个节点
        tab[i] = newNode(hash, key, value, null);//当前桶为空，直接插入节点
    else {//当前桶不为空
        Node<K,V> e; K k;//注意这个e，代表桶中与key对应的节点，e!=null则表示要插入的键值对已经存在。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))//p就是key对应的节点
            e = p;
        else if (p instanceof TreeNode)//p是红黑树节点，根据红黑树相关方法找到key对应节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {//p是普通链表节点
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {//当前桶中无key对应节点则插入新节点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) //当前桶中数量大于等于TREEIFY_THRESHOLD则转为红黑树存储
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))//找到key对应节点，退出循环
                    break;
                p = e;
            }
        }
        if (e != null) { // 要插入的键值对已经存在，更新value，并返回旧的value
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);//修改之后的回调，HashMap中没有任何操作。
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)//检查扩容
        resize();
    afterNodeInsertion(evict);//插入之后的回调，HashMap中没有任何操作。
    return null;
}
```

### get方法

```java
/**  
 * 返回HashMap中给定key对应的value，没有则返回null。  
 *  
 * 严格来说, 如果HashMap中有任意一对键值对key为k，value为v，使(key==null ? k==null :   
 * key.equals(k))为true那么返回v，否则返回null。  
 *  
 * 返回值为null并不一定代表HashMap中没有给定key的键值对，也可能是给定key对应的value就是null。可以使用  
 * containsKey方法来区分这两种情况。  
 *  
 * @see #put(Object, Object)  
 */
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

/**  
 * Map.get及相关方法的实现。  
 *  
 * @param hash key的hash值。  
 * @param key 指定的key。  
 * @return 返回key对应的键值对，没有则返回null。  
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {//桶数组及key哈希映射对应的桶不为空则继续查找。
        if (first.hash == hash && // 总是检查第一个节点，是要查找的则直接返回。
            ((k = first.key) == key || (key != null && key.equals(k))))//对应key相同就返回。
            return first;
        if ((e = first.next) != null) {//不为空则查找后续节点。
            if (first instanceof TreeNode)//红黑树节点则调用对应方法查找。
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {//普通链表则循环查找。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

###remove方法

```java
/**  
 * 如果给定key对应的键值对存在则删除该键值对。  
 *  
 * @param  key 要删除的键值对的key。  
 * @return 返回旧的value值，如果不存在则返回null。返回值为null不一定表示key对应的键值对不存在，也可能是旧  
 * 的value为null。  
 */
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

/**  
 * Map.remove及相关方法的实现。  
 *  
 * @param hash key的hash值  
 * @param key 要操作的key  
 * @param value 如果matchValue为false则没有意义。  
 * @param matchValue 如果是true则只有在key对应的value与给定value相同才删除。  
 * @param movable 如果是false则删除时不移动其他节点。（红黑树节点专用）  
 * @return 返回key对应节点，不存在则返回null。  
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;//注意这个p
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {//桶数组及key哈希映射对应的桶不为空则继续查找。
        Node<K,V> node = null, e; K k; V v;//如果查到要删除的节点，这个node就指向要删除的节点。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))//第一个节点就是要删除的。
            node = p;//p和node指向要删除的节点。
        else if ((e = p.next) != null) {//第一个不是，继续查找。
            if (p instanceof TreeNode)//红黑树节点则调用对应方法查找。
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {//普通链表则循环查找。
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;//p指向要删除的节点的前一个节点。
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)//要删除的节点为红黑树节点则调用相关方法删除。
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)//p和node都指向要删除的节点（首节点），首节点指向原首节点的下一个节点。
                tab[index] = node.next;
            else//p指向要删除的节点的前一个节点，将p的next指向要删除的节点的next。
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);//删除节点回调方法，HashMap中无操作。
            return node;
        }
    }
    return null;
}
```

### resize方法

```java
/**  
 * 初始化或翻倍HashMap的大小.  如果table为null根据threshold或默认值代表的初始容量来分配内存。否则，节点   
 * 要么存储在原索引位置，要么移动指定的偏移量后存储到新table中。  
 *  
 * @return the table  
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {//oldCap > 0代表是扩容即发生了rehash
        if (oldCap >= MAXIMUM_CAPACITY) {//oldCap达到最大值，threshold赋值为int最大值，不rehash
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)//容量翻倍后小于最大值且oldCap大于等于默认初始容量（这里oldCap要大于DEFAULT_INITIAL_CAPACITY？）
            newThr = oldThr << 1; // newThr翻倍
    }
    else if (oldThr > 0) // 未初始化且手动设置了初始容量，threshold就是初始容量（此时为初始化）
        newCap = oldThr;
    else {               // oldCap和threshold都为0代表使用默认值（此时为初始化操作）
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {//计算新的threshold
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//创建新的桶数组
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)//当前桶中只有一个节点（即没发生hash冲突）
                    newTab[e.hash & (newCap - 1)] = e;//重新计算索引并插入新位置
                else if (e instanceof TreeNode)//如果是红黑树节点则调用split方法处理
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 普通链表节点，保持原来的顺序
                    Node<K,V> loHead = null, loTail = null;//存储在旧空间
                    Node<K,V> hiHead = null, hiTail = null;//存储到新扩展的空间
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
//这里由于HashMap扩容是扩大2倍，所以(e.hash & oldCap) == 0则节点e仍存储与原位置即可（等同于存储到e.hash & （newCap - 1））
                            if (loTail == null)//指向头节点，方便赋值
                                loHead = e;
                            else
                                loTail.next = e;//保留原顺序
                            loTail = e;
                        }
                        else {
//(e.hash & oldCap) != 0则节点e要存储到原位置加偏移量oldCap（等同于存储到e.hash & （newCap - 1））
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {//指向头节点
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {//指向头节点
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```