# HashMap源码阅读
> 本文基于JDK1.8  >读完本文预计需要25分钟(因有大量源代码，电脑屏观看体验较佳)
## 摘要
HashMap相信这是出现频率最高的面试点之一，应该是面试问到烂的面试题之一，同时也是Java中用于处理键值对最常用的数据类型。那么我们就针对JDK8的HashMap共同学习一下！
## 主要方法
### 关键变量：
```java
    /**
     * The default initial capacity - MUST be a power of two.
     * 初始容量大小 必须是2的次幂
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     * 最大的容量大小
     * 超过这个值就将threshold修改为Integer.MAX_VALUE，数组不进行扩容
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * 负载因子 为什么是0.75？因为统计学中hash冲突符合泊松分布，7-8之间冲突最小
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     * 链表大于这个值就会树化
     * 注意：树化并不是整个map链表，而是某一个大于此阈值的链表
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     * 小于这个值就会反树化
     */
    static final int UNTREEIFY_THRESHOLD = 6;
```

### 四个构造方法：
```java
//构造方法1
//指定初始容量大小，负载因子
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY) //1<<30 最大容量是 Integer.MAX_VALUE;
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        //tableSizeFor这个方法用于找到大于等于initialCapacity的最小的2的幂
        this.threshold = tableSizeFor(initialCapacity);
    }

//构造方法2
//其实调用了上边的构造方法1 负载因子给的默认值0.75
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

//构造方法3
//空参构造，均使用默认值
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    
//构造方法4
//与其他三个相比，初始化了
public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;//0.75f
        //调用了putVal方法，而putVal方法中有resize方法，有初始化
        putMapEntries(m, false);
    }

```
对四个构造方法简单总结一下：

    1、前三个构造函数并没有初始化，都是用到的时候去初始化
    
    2、构造方法4相当于用到了put方法，所以初始化了


### hashmap->hash()
```java
/**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     * 计算key.hashCode（）并将哈希的较高位（XOR）扩展为较低。
     * 由于该表使用2的幂次掩码，因此仅在当前掩码上方的位中发生变化的哈希集将始终发生冲突。
     * （众所周知的示例是在小表中包含连续整数的Float键集。）因此，我们应用了一种变换，
     * 将向下扩展较高位的影响。 在速度，实用性和位扩展质量之间需要权衡。
     * 由于许多常见的哈希集已经合理分布（因此无法从扩展中受益），
     * 并且由于我们使用树来处理容器中的大量冲突，因此我们仅以最便宜的方式对一些移位后的位进行XOR，
     * 以减少系统损失， 以及合并最高位的影响，否则由于表范围的限制，这些位将永远不会在索引计算中使用。
     */
    static final int hash(Object key) {
        int h;
        //为什么^ (h >>> 16) 更加散列,性能考量，方便位运算
        //方便在put,get方法中(n - 1) & hash计算数组下标
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
        //HashMap中key值可以为null, 看到0那么我们可以判断null值一定存储在数组的第一个位置
    }

```




### hashmap->put()

主要逻辑：

![](https://imgkr2.cn-bj.ufileos.com/4ec1bf7a-d545-4959-8969-34475b998a5d.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=6UM5nQ1FdqIHoeXzB9gHkScqabc%253D&Expires=1611498422)


以下是源代码（带注释）： 
```java

    /**
         * Associates the specified value with the specified key in this map.
         * If the map previously contained a mapping for the key, the old
         * value is replaced.
         * //将指定的值与此映射中的指定键关联。如果该映射先前包含了该键的映射，则旧值将被替换。
         *
         * @param key key with which the specified value is to be associated
         * @param value value to be associated with the specified key
         * @return the previous value associated with <tt>key</tt>, or
         *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
         *         (A <tt>null</tt> return can also indicate that the map
         *         previously associated <tt>null</tt> with <tt>key</tt>.)
         */
        public V put(K key, V value) {
            //把key先去hash一下拿到hash值
            return putVal(hash(key), key, value, false, true);
        }
    
        /**
         * Implements Map.put and related methods.
         *
         * @param hash hash for key
         * @param key the key
         * @param value the value to put
         * @param onlyIfAbsent if true, don't change existing value //if true 则不改变原来存在的值
         * @param evict if false, the table is in creation mode.//if false 则表处于创建模式
         * @return previous value, or null if none
         */
        final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
    
            //数组+链表+红黑树，链表型（Node泛型)数组，每一个元素代表一条链表，则每个元素称为桶
            //HashMap 的每一个元素，都是链表的一个节点（Entry<K,V>）这里也就是Node<K,V>
    
            //tab:桶 p:桶 n:哈希表数组大小 i:数组下标(桶的位置)
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            //1.判断当前桶是否为空，空的就调用resize()方法（resize 中会判断是否进行初始化）
            if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length;
            //2.判断是否有hash冲突，根据入参key与key的hash值找到具体的桶并判空，空则无冲突 直接新建桶
            //？为什么采用(n - 1) & hash计算数组下标，感兴趣的可以深入了解
            if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
            //3.以下表示有冲突，处理hash冲突
            else {
                Node<K,V> e; K k;//均为临时变量
                //4.判断当前桶的key是否与入参key一致，一致则存在，把当前桶p赋值给e,覆盖原 value 在步骤10进行
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                //5.如果当前的桶为红黑树，用putTreeVal()方法写入 赋值给e
                else if (p instanceof TreeNode)
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                //6.则当前的桶是链表 遍历链表
                else {
                    for (int binCount = 0; ; ++binCount) {
                        if ((e = p.next) == null) {
                            //7.尾插法，链表下一个节点是null（链表末尾），就new一个新节点写入到当前链表节点的后面
                            p.next = newNode(hash, key, value, null);
                            //8.判断是否大于阈值，需要链表转红黑树
                            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                                //binCount从0开始的, 所以当binCount为7时，链表长度为8（算上数组槽位开始的那个节点，总长度为9），则需要树化桶
                                treeifyBin(tab, hash);
                            break;
                        }
                        //9.与步骤4一致，如果链表中key存在则直接跳出 步骤10覆盖原值
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        p = e;
                    }
                }
                //10.存在相同的key的Node节点,则覆盖原value
                if (e != null) { // existing mapping for key
                    V oldValue = e.value;
                    //onlyIfAbsent为true：不改变原来的值 ；false： 改变原来的值
                    if (!onlyIfAbsent || oldValue == null)
                        e.value = value;
                    //LinkedHashMap用到的回调方法
                    afterNodeAccess(e);
                    return oldValue;
                }
            }
             /*记录修改次数标识
             用于fast-fail，由于HashMap非线程安全，在对HashMap进行迭代时，
             如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），
             需要抛出异常ConcurrentModificationException
             */
            ++modCount;
            //11.容量超过阈值，扩容
            if (++size > threshold)
                resize();
            //LinkedHashMap用到的回调方法
            afterNodeInsertion(evict);
            return null;
        }

```

那么我们来总结一下put方法：
```
1、开始，入参key、value
2、判断当前table是否为空或者length=0？
    是，去扩容，（resize()方法中有判断是否初始化）
    否，根据key算出hash值并得到插入的数组的索引
        判断找到的这个table[i]是否为空？
            是，直接插入，再到步骤
            否，判断key是否存在？
                是，直接覆盖对应的value,再到步骤3
                否，去判断当前这个table[i]是不是treeNode？
                    是，使用红黑树的方式插入key、value
                    否，开始遍历链表准备插入
                        判断链表长度是不是大于8？
                            是，链表转红黑树插入key、value
                            否，以链表的方式插入key、value如果key存在就直接覆盖对应value

3、判断map的size()是否大于阈值？
	    是 就去扩容resize()
4、结束
```

### hashmap->get()
```java

public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //1、判断当前数组不为空并长度大于0 && 由key的hash值找到对应数组下的桶（可能是红黑树或者链表）
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //2、先判断桶的第一个节点 如果key一致 返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //3、再判空下一个节点不为空 && 判断是红黑树还算链表
            if ((e = first.next) != null) {
                //4、如果是红黑树 则按红黑树方式取值
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //否则就是链表，遍历取值
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```
那么我们来总结一下get()方法：
```$xslt
1、开始，入参key
2、判断当前的数组长度不为空&&length>0
    是 return null;
    否，去判断第一个节点，如果key符合，返回
        再去判断下一个节点是否为空
            是，return null
            否，判断否是红黑树？
                是，按红黑树的方式取值
                否，遍历链表取值

3、结束
```

### hashmap->resize()

```java
/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *初始化或增加表大小。 如果为空，则根据字段阈值中保持的初始容量目标进行分配。
     * 否则，因为我们使用的是2的幂，所以每个bin中的元素必须保持相同的索引，或者在新表中以2的幂偏移。
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //1、原数组扩容
        if (oldCap > 0) {
            //如果原数组长度大于最大容量，把阈值调最大，return
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //把原数组大小、阈值都扩大一倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //使用了指定initialCapacity的构造方法，则用原阈值作为新容量
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //使用空参构造，用默认值
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;//16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//0.75*16=12
        }
        //使用了指定initialCapacity的构造方法，新阈值为0，则计算新的阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //2、用新的数组容量大小初始化数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //如果仅仅是初始化过程,到此结束 return newTab
        table = newTab;
        //3、开始扩容的主要工作，数据迁移
        if (oldTab != null) {
            //遍历原数组开始复制旧数据
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;//清除旧表引有
                    //原数组中单个元素，直接复制到新表
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果该元素类型是红黑树，按红黑树方式处理
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //这段代码设计巧妙，环环相扣啊
                        //先定义了两种类型的链表 以及头尾节点 高位链表与低位链表
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        //按顺序遍历原链表的节点
                        do {
                            next = e.next;
                            //这是一个核心的判断条件，感兴趣的可以深入了解？为什么这么做
                            //=0则放到低位链表
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //否则放到高位链表
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                            //以上实际上就是对原来链表拆分成了两个高低位链表
                        } while ((e = next) != null);
                        //把整个低位链表放到新数组的j位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //把整个高位链表放到新数组的j+oldCap位置上
                        if (hiTail != null) {
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

总结一下resize()方法：
```java
1、开始，拿到原数组
2、对原数组扩容
	  2.1如果原数组中的容量到最大，不再扩容，return原数组
	  2.2把原数组容量大小与阈值都扩大一倍
3、如初始化用的指定initialCapacity的构造方法，则用原阈值作为新容量
4、如初始化时候用的空参构造，用默认容量与默认阈值
5、如初始化用的指定initialCapacity的构造方法，阈值=0，计算新的阈值
6、用新的容量初始化数组，如果是初始化，结束返回新数组
7、开始扩容，做数据迁移
	  7.1遍历原数组copy数据到新数组
		    7.1.1如数组中只有一个元素，则直接复制
		    7.1.2如元素是红黑树数类型，则按红黑树的方式处理
		    7.1.3对原数组的链表进行处理
				      定义一个高位链表、一个低位链表（对原链表拆分）
				      开启一个循环，遍历原链表
					        判断条件e.hash & oldCap == 0?
						          是，把这些链表节点放到低位链表
						          否，放到高位链表
				      循环结束，遍历链表完成
				      把整个低位链表放到新数组j位置
				      把整个高位链表放到新数组j+oldCap位置
	  7.2循环结束，遍历旧数组完成
8、返回新数组
```
我想这会阅读结束之后应该对HashMap有了一定的认识，希望能在面试或者工作中帮到您！

## Hongchen闲谈

![](https://static01.imgkr.com/temp/2164c217bd6d493ab98813818390318f.jpg)

看起来是不是有点酷，它的名字叫”茑屋“，北京温榆河公园。

上周跟女友在家待闷了出门转转，于是我们找了个公园，秋冬的公园还是比较萧瑟。

年底了，疫情零零散散，希望早点过去！peaceeeee

## 感谢阅读
才疏学浅，如有你发现有错误的地方，可以在后台或者评论提出来，我会加以修改。
感谢您的阅读，欢迎并感谢关注！

![](https://imgkr2.cn-bj.ufileos.com/5112667e-581e-4239-bf60-ac118fd7df28.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=EsE8dDDl2DoHCNlBOLID1PR3tXc%253D&Expires=1611500625)
