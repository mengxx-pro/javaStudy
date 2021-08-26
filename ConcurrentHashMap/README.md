# JDK1.8 ConcurrentHashMap 支持多线程操作

没有segement对象(分段锁)

用的是Node数组

扩容的时候，每个线程有转移的区间(步长)，加锁加的是node[index]

**// 构造方法没有参数，什么都不干**

        public ConcurrentHashMap() {
        }

//用来计算数组长度，sizeCtl 的值是2的幂等数

    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }


**Put的主流程**
    
    final V putVal(K key, V value, boolean onlyIfAbsent) {
       //不能put null,空字符串可以
        if (key == null || value == null) throw new NullPointerException();
        // 得到 hash 值
        int hash = spread(key.hashCode());
        // 用于记录相应链表的长度
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果数组"空"，进行数组初始化
            if (tab == null || (n = tab.length) == 0)
                // 初始化数组，后面会详细介绍
                tab = initTable();

        // 找该 hash 值对应的数组下标，得到第一个节点 f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果数组该位置为空，
            //    用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
            //          如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上也能猜到，肯定是因为在扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);

        else { // 到这里就是说，f 是该位置的头结点，而且不为空

            V oldVal = null;
            // 获取数组该位置的头结点的监视器锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 头结点的 hash 值大于 0，说明是链表
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果发现了"相等"的 key，判断是否要进行值覆盖，然后也就可以 break 了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面，尾插法
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }

            if (binCount != 0) {
                // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树???
                    //    具体源码我们就不看了，扩容部分后面说
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 
    addCount(1L, binCount);
    return null;
    }


casTabAt是赋值放到table[]

MOVED=-1代表当前正在扩容，helpTransfer帮助扩容，

put的时候用synchronized加锁，对node上的第一个元素加锁，

然后tabAt(tab,i)==f判断现在node的头节点还是不是f，看有没有其他线程对f进行了操作

然后判断是node还是treeBin

node：

 尾插法

treeBin：

判断元素个数是否大于8，转成红黑树

Synchronized加锁，改成双向链表，

TreeBin对象(是为了方便加锁？直接锁TreeBin？)，里边包括红黑树TreeNode


**初始化initTable：**

Sc =sizeCtl默认等于0

sc先减1，通过compareAndSwapInt cas的方式操作

sc=n-(n>>>2); =n-1/4*n =0.75

Volatile表示从内存里拿出来

Thread.yield();先放弃CPU的资源，

    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 初始化的"功劳"被其他线程"抢去"了
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // CAS 一下，将 sizeCtl 设置为 -1，代表抢到了锁
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // DEFAULT_CAPACITY 默认初始容量是 16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        // 初始化数组，长度为 16 或初始化时提供的长度
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        // 将这个数组赋值给 table，table 是 volatile 的
                        table = tab = nt;
                        // 如果 n 为 16 的话，那么这里 sc = 12
                        // 其实就是 0.75 * n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 设置 sizeCtl 为 sc，我们就当是 12 吧
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }


**//put里最后判断元素个数是否转红黑树的treeifyBin**

        private final void treeifyBin(Node<K,V>[] tab, int index) {
            Node<K,V> b; int n, sc;
            if (tab != null) {
                // MIN_TREEIFY_CAPACITY 为 64
                // 所以，如果数组长度小于 64 的时候，其实也就是 32 或者 16 或者更小的时候，会进行数组扩容
                if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                    // 后面我们再详细分析这个方法
                    tryPresize(n << 1);
                // b 是头结点
                else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                    // 加锁
                    synchronized (b) {
        
                        if (tabAt(tab, index) == b) {
                            // 下面就是遍历链表，建立一颗红黑树
                            TreeNode<K,V> hd = null, tl = null;
                            for (Node<K,V> e = b; e != null; e = e.next) {
                                TreeNode<K,V> p =
                                    new TreeNode<K,V>(e.hash, e.key, e.val,
                                                      null, null);
                                if ((p.prev = tl) == null)
                                    hd = p;
                                else
                                    tl.next = p;
                                tl = p;
                            }
                            // 将红黑树设置到数组相应位置中
                            setTabAt(tab, index, new TreeBin<K,V>(hd));
                //new TreeBin 会对红黑树进行整体的重新排列节点等
                        }
                    }
                }
            }
        }




**扩容：**

baseCount基数大小

CounterCell{] 统计元素的个数（计数器），先对baseCount用cas加一，不行的就对CounterCell加+1

U.compareAndSwapLong(this,BASECOUNT,b=baseCount,s=b+x)是对baseCount+1

每个线程生成随机数ThreadLocalRandom.getProbe()，


**数据迁移：transfer**

下面这个方法有点长，将原来的 tab 数组的元素迁移到新的 nextTab 数组中。

虽然我们之前说的 tryPresize 方法中多次调用 transfer 不涉及多线程，但是这个 transfer 方法可以在其他地方被调用，典型地，我们之前在说 put 方法的时候就说过了，请往上看 put 方法，是不是有个地方调用了 helpTransfer 方法，helpTransfer 方法会调用 transfer 方法的。

此方法支持多线程执行，外围调用此方法的时候，会保证第一个发起数据迁移的线程，nextTab 参数为 null，之后再调用此方法的时候，nextTab 不会为 null。

阅读源码之前，先要理解并发操作的机制。原数组长度为 n，所以我们有 n 个迁移任务，让每个线程每次负责一个小任务是最简单的，每做完一个任务再检测是否有其他没做完的任务，帮助迁移就可以了，而 Doug Lea 使用了一个 stride，简单理解就是步长，每个线程每次负责迁移其中的一部分，如每次迁移 16 个小任务。所以，我们就需要一个全局的调度者来安排哪个线程执行哪几个任务，这个就是属性 transferIndex 的作用。

第一个发起数据迁移的线程会将 transferIndex 指向原数组最后的位置，然后从后往前的 stride 个任务属于第一个线程，然后将 transferIndex 指向新的位置，再往前的 stride 个任务属于第二个线程，依此类推。当然，这里说的第二个线程不是真的一定指代了第二个线程，也可以是同一个线程，这个读者应该能理解吧。其实就是将一个大的迁移任务分为了一个个任务包。
    
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
            int n = tab.length, stride;
        //stride步长
          // stride 在单核下直接等于 n，多核模式下为 (n>>>3)/NCPU，最小值是 16
            // stride 可以理解为”步长“，有 n 个位置是需要进行迁移的，
            //   将这 n 个任务分为多个任务包，每个任务包有 stride 个任务
            if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
                stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 如果 nextTab 为 null，先进行一次初始化
        //    前面我们说了，外围会保证第一个发起迁移的线程调用此方法时，参数 nextTab 为 null
        //       之后参与迁移的线程调用此方法时，nextTab 不会为 null
    
             if (nextTab == null) {            // initiating
                try {
                    @SuppressWarnings("unchecked")
            //容量翻倍
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                    nextTab = nt;
                } catch (Throwable ex) {      // try to cope with OOME
                    sizeCtl = Integer.MAX_VALUE;
                    return;
                }
                nextTable = nextTab;
                transferIndex = n;//老元素的数组长度
            }
            int nextn = nextTab.length;
    
        //ForwardingNode表明老数组正在扩容，会帮助转移？
          // ForwardingNode 翻译过来就是正在被迁移的 Node
            // 这个构造方法会生成一个Node，key、value 和 next 都为 null，关键是 hash 为 MOVED
            // 后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
            //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
            //    所以它其实相当于是一个标志。
        
            ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
            boolean advance = true;//是否前进，继续寻找其他主要转移的元素
            boolean finishing = false; // to ensure sweep before committing nextTab//当前线程是否还有需要做的
    
     /*
         * 下面这个 for 循环，最难理解的在前面，而要看懂它们，应该先看懂后面的，然后再倒回来看
         * 
         */
    
        // i 是位置索引，bound 是边界，注意是从后往前
            for (int i = 0, bound = 0;;) {
                Node<K,V> f; int fh;
          //   简单理解结局：i 指向了 transferIndex，bound 指向了 transferIndex-stride
                while (advance) {
                    int nextIndex, nextBound;
                    if (--i >= bound || finishing)
                        advance = false;
    
                  // 将 transferIndex 值赋给 nextIndex
                  // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
                    else if ((nextIndex = transferIndex) <= 0) {
                        i = -1;
                        advance = false;
                    }
            //nextIndex开始是数组长度，stride是上边计算的，bound是控制区域的第一个位置，
            //从后向前，nextIndex在这里被改了
                    else if (U.compareAndSwapInt
                             (this, TRANSFERINDEX, nextIndex,
                              nextBound = ( > stride ?
                                           nextIndex - stride : 0))) {
                       // 看括号中的代码，nextBound 是这次迁移任务的边界，注意，是从后往前
                        bound = nextBound;
                        i = nextIndex - 1;
                        advance = false;
                    }
                }
                if (i < 0 || i >= n || i + n >= nextn) {
                    int sc;
                    if (finishing) {
             //全部转移完成table才会修改
            // 所有的迁移操作已经完成
                        nextTable = null;
                    // 将新的 nextTab 赋值给 table 属性，完成迁移
                        table = nextTab;
                    // 重新计算 sizeCtl：n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                        sizeCtl = (n << 1) - (n >>> 1);
                        return;
                    }
            //sc=sizeCtl说明扩容完成
               // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
               // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
               // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            
                    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 任务结束，方法退出
                        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                            return;
    
                        // 到这里，说明 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，
                        // 也就是说，所有的迁移任务都做完了，也就会进入到上面的 if(finishing){} 分支了
                        finishing = advance = true;
                        i = n; // recheck before commit
                    }
                }
              // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
                else if ((f = tabAt(tab, i)) == null)
            //fwd赋值给i
                    advance = casTabAt(tab, i, null, fwd);
            // 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
                else if ((fh = f.hash) == MOVED)
                    advance = true; // already processed
                else {
                // 对数组该位置处的结点加锁，开始处理数组该位置处的迁移工作
            //开始转移，加锁，加到node里
                    synchronized (f) {
                        if (tabAt(tab, i) == f) {
                            Node<K,V> ln, hn;
                        // 头结点的 hash 大于 0，说明是链表的 Node 节点
                // 下面这一块和 Java7 中的 ConcurrentHashMap 迁移是差不多的，
                            // 需要将链表一分为二，
                            //   找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                            //   lastRun 之前的节点需要进行克隆，然后分到两个链表中
    
                            if (fh >= 0) {
                                int runBit = fh & n;
                                Node<K,V> lastRun = f;
                                for (Node<K,V> p = f.next; p != null; p = p.next) {
                                    int b = p.hash & n;
                                    if (b != runBit) {
                                        runBit = b;
                                        lastRun = p;
                                    }
                                }
                                if (runBit == 0) {
                                    ln = lastRun;
                                    hn = null;
                                }
                                else {
                                    hn = lastRun;
                                    ln = null;
                                }
                                for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                    int ph = p.hash; K pk = p.key; V pv = p.val;
                                    if ((ph & n) == 0)
                                        ln = new Node<K,V>(ph, pk, pv, ln);
                                    else
                                        hn = new Node<K,V>(ph, pk, pv, hn);
                                }
                  //node里分2部分的元素转移
                    // 其中的一个链表放在新数组的位置 i
                                setTabAt(nextTab, i, ln);
                            // 另一个链表放在新数组的位置 i+n
                                setTabAt(nextTab, i + n, hn);
                           // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                            //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                                setTabAt(tab, i, fwd);
                            // advance 设置为 true，代表该位置已经迁移完毕
                                advance = true;
                            }
                            else if (f instanceof TreeBin) {
                            // 红黑树的迁移
                                TreeBin<K,V> t = (TreeBin<K,V>)f;
                                TreeNode<K,V> lo = null, loTail = null;
                                TreeNode<K,V> hi = null, hiTail = null;
                                int lc = 0, hc = 0;
                                for (Node<K,V> e = t.first; e != null; e = e.next) {
                                    int h = e.hash;
                                    TreeNode<K,V> p = new TreeNode<K,V>
                                        (h, e.key, e.val, null, null);
                                    if ((h & n) == 0) {
                                        if ((p.prev = loTail) == null)
                                            lo = p;
                                        else
                                            loTail.next = p;
                                        loTail = p;
                                        ++lc;
                                    }
                                    else {
                                        if ((p.prev = hiTail) == null)
                                            hi = p;
                                        else
                                            hiTail.next = p;
                                        hiTail = p;
                                        ++hc;
                                    }
                                }
                            // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                                ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                    (hc != 0) ? new TreeBin<K,V>(lo) : t;
                                hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                    (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            // 将 ln 放置在新数组的位置 i
                                setTabAt(nextTab, i, ln);
                            // 将 hn 放置在新数组的位置 i+n
                                setTabAt(nextTab, i + n, hn);
                           // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                            //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                                setTabAt(tab, i, fwd);
                            // advance 设置为 true，代表该位置已经迁移完毕
                                advance = true;
                            }
                        }
                    }
                }
            }
        }

说到底，transfer 这个方法并没有实现所有的迁移任务，每次调用这个方法只实现了 transferIndex 往前 stride 个位置的迁移工作，其他的需要由外围来控制。


**最后addCount统计数组里元素个数**

//check在remove方法中传的是-1

 addCount(long x, int check) 

fullAddCount对cell数组里加1，可以加在basecout，或者countcell

collide冲突的意思
    
    private final void addCount(long x, int check) {
            CounterCell[] as; long b, s;
            if ((as = counterCells) != null ||
                !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
                CounterCell a; long v; int m;
                boolean uncontended = true;
                if (as == null || (m = as.length - 1) < 0 ||
                    (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                    !(uncontended =
                      U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                    fullAddCount(x, uncontended);
                    return;
                }
                if (check <= 1)
                    return;
                s = sumCount();
            }
            if (check >= 0) {
                Node<K,V>[] tab, nt; int n, sc;
                while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                       (n = tab.length) < MAXIMUM_CAPACITY) {
                    int rs = resizeStamp(n);
                    if (sc < 0) {
                        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                            sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                            transferIndex <= 0)
                            break;
                        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                            transfer(tab, nt);
                    }
                    else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                                 (rs << RESIZE_STAMP_SHIFT) + 2))
                        transfer(tab, null);
                    s = sumCount();
                }
            }
        }
    
    
    private final void fullAddCount(long x, boolean wasUncontended) {
    {
            int h;
            if ((h = ThreadLocalRandom.getProbe()) == 0) {
                ThreadLocalRandom.localInit();      // force initialization
                h = ThreadLocalRandom.getProbe();
                wasUncontended = true;
            }
            boolean collide = false;                // True if last slot nonempty
            for (;;) {
                CounterCell[] as; CounterCell a; int n; long v;
                if ((as = counterCells) != null && (n = as.length) > 0) {
                    if ((a = as[(n - 1) & h]) == null) {
                        if (cellsBusy == 0) {            // Try to attach new Cell
                            CounterCell r = new CounterCell(x); // Optimistic create
                            if (cellsBusy == 0 &&
                                U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                                boolean created = false;
                                try {               // Recheck under lock
                                    CounterCell[] rs; int m, j;
                                    if ((rs = counterCells) != null &&
                                        (m = rs.length) > 0 &&
                                        rs[j = (m - 1) & h] == null) {
                                        rs[j] = r;
                                        created = true;
                                    }
                                } finally {
                                    cellsBusy = 0;
                                }
                                if (created)
                                    break;
                                continue;           // Slot is now non-empty
                            }
                        }
                        collide = false;
                    }
                    else if (!wasUncontended)       // CAS already known to fail
                        wasUncontended = true;      // Continue after rehash
                    else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                        break;
                    else if (counterCells != as || n >= NCPU)
                        collide = false;            // At max size or stale
                    else if (!collide)
            //扩容的时候用
                        collide = true;
                    else if (cellsBusy == 0 &&
                             U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        try {
                            if (counterCells == as) {// Expand table unless stale
                //2次循环，cas加一都不成功就 扩容
                                CounterCell[] rs = new CounterCell[n << 1];
                                for (int i = 0; i < n; ++i)
                                    rs[i] = as[i];
                                counterCells = rs;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        collide = false;
                        continue;                   // Retry with expanded table
                    }
                    h = ThreadLocalRandom.advanceProbe(h);
                }
        //cellsBusy当前数组有没有其他线程在使用，0是没有在使用
        //CELLSBUSY改成1，表明在使用这个数组
                else if (cellsBusy == 0 && counterCells == as &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    boolean init = false;
                    try {                           // Initialize table
                        if (counterCells == as) {
                            CounterCell[] rs = new CounterCell[2];
                //cell数组初始化，h是随机数
                            rs[h & 1] = new CounterCell(x);
                            counterCells = rs;
                            init = true;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    if (init)
                        break;
                }
        //对cells操作不了的，用过这个cas对basecount++
                else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                    break;                          // Fall back on using base
            }
        }
    
