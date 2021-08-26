# JDK1.8  HashMap 

**1.8里  hashmap是数组+链表+红黑树**

红黑树的时间复杂度可以降低为 O(logN)

黑红数是平衡的(插入，查询时间复杂度比较好 logn)

红黑树定义：

每个节点是红色或者黑色

根节点是黑色的，叶子节点是黑色的(指最后的节点，空的那些节点)

如果一个节点是红色的，那他子节点是黑色的

对每个结点，到每个子孙叶子节点，包括相同的黑色节点

插入的过程：

 新插入的节点都是红色的

 如果父节点是黑色的不用调整

 如果父节点是红色的：

   1.叔叔是空的，旋转+变色(本身节点是右边的，叔叔是空的，要经过叔叔节点左旋，祖父节点右旋的步骤)
  
   2.叔叔节点是红色的，父节点+叔叔 变色，祖父节点变色

   3.叔叔节点是黑色的，左旋+变色

**初始化**

初始化的大小也是16，2的幂等函数，这样在对hash值位与运算(length-1 二进制就是0000 1111 )只有后几位是和key

的值相关的，只要key是均匀的生成的index就是均匀的，这是为了实现均匀分布

**put的过程**

设置初始化容量n，初始化threshold = 大于n数且最接近的2的整数次幂的数 * 负载因子

初始化的时候是不分配node数组大小的，put的时候如果数组是空的首先会进行resize()操作，

这里的resize就不是扩容了，是初始化数组(初始化到默认的 16 或自定义的初始容量)


**_put的时候先看是node还是红黑树_**

_如果是node_

node的依次遍历链表查看有无key的值，有的话更新，没有的话进行尾插法

Node在插入的时候采用尾插法，1.7的时候是头插法，头插法会改变元素之间的顺序，

在扩容的时候迁移元素的有可能会出现循环链表，导致出现问题，而尾插法不会改变元素之间的

顺序，这样迁移的时候还是按照之前的顺序来，不会出现循环链表

Java7在多线程操作HashMap时可能引起死循环，原因是扩容转移后前后链表顺序倒置，在转移过程中修改了原来链表中节点的引用关系。

Java8在同样的前提下并不会引起死循环，原因是扩容转移后前后链表顺序不变，保持之前节点的引用关系。

_如果是红黑树(过程如下node转红黑树里的比较节点大小，balanceInsertion调节节点位置颜色，moveRootToFront移动节点到头部)_

_如果是node但是大于8个了_

   判断node数组长度是否小于64，小于的话直接扩容(先插入后转换)，不用转红黑树(扩容的话数组长度变大，链表的长度就会缩小，提升插入效率)
	
   大于64了就要转成红黑树

   node转成红黑树过程treeify：

   1.依次转成treenode，treenode是有依次双向指向的(prev和next互指)

   2. 然后treeify方法里循环判断当前值的哈希值与root节点的哈希值比较，把他放到合适的位置，再调用balanceInsertion
	
  来调整对应子树的颜色，节点位置等(变色+左旋+右旋)
	
	x是新节点，xp是父节点，xpp是祖父节点
	  static<K,V>TreeNode<K,V>balanceInsertion(TreeNode<K,V>root,
	      TreeNode<K,V>x){
	    x.red=true;
	    for(TreeNode<K,V>xp,xpp,xppl,xppr;;){
	      //没有父节点，说明这是第一个节点
	      if((xp=x.parent)==null){
	        x.red=false;
	        returnx;
	      }
	      //父节点是黑色的不用调整，或者没有祖父节点的也不用调整
	      elseif(!xp.red||(xpp=xp.parent)==null)
	      Return root;
	      if(xp==(xppl=xpp.left)){
	        if((xppr=xpp.right)!=null&&xppr.red){
	          xppr.red=false;
	          xp.red=false;
	          xpp.red=true;
	          x=xpp;
	        }
	        else{
	          if(x==xp.right){
	            root=rotateLeft(root,x=xp);
	            xpp=(xp=x.parent)==null?null:xp.parent;
	          }
	          if(xp!=null){
	            xp.red=false;
	            if(xpp!=null){
	              xpp.red=true;
	              root=rotateRight(root,xpp);
	            }
	          }
	        }
	      }
	      else{
	        if(xppl!=null&&xppl.red){
	          xppl.red=false;
	          xp.red=false;
	          xpp.red=true;
	          x=xpp;
	        }
	        else{
	          if(x==xp.left){
	            root=rotateRight(root,x=xp);
	            xpp=(xp=x.parent)==null?null:xp.parent;
	          }
	          if(xp!=null){
	            xp.red=false;
	            if(xpp!=null){
	              xpp.red=true;
	              root=rotateLeft(root,xpp);
	            }
	          }
	        }
	      }
	    }
	  }
	
	
   3. moveRootToFront方法里会把老的node[]中的根节点拿到第一个（头部）？没搞懂为啥呀这样
	（红黑树里也隐藏了node属性）
	

**扩容**

    // 初始化或者扩容之后元素调整当元素个数达到初始化总数*扩容因子就会扩容
	final Node<K,V>[] resize() {
		// 获取旧元素数组的各种信息
	    Node<K,V>[] oldTab = table;
	    // 长度
	    int oldCap = (oldTab == null) ? 0 : oldTab.length;
	    // 扩容的临界值
	    int oldThr = threshold;
	    // 定义新数组的长度及扩容的临界值
	    int newCap, newThr = 0;
	    // 如果原table不为空
	    if (oldCap > 0) {
	    	// 如果数组长度达到最大值，则修改临界值为Integer.MAX_VALUE
	        if (oldCap >= MAXIMUM_CAPACITY) {
	            threshold = Integer.MAX_VALUE;
	            return oldTab;
	        }
	        // 下面就是扩容操作（2倍）
	        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
	                 oldCap >= DEFAULT_INITIAL_CAPACITY)
	            // threshold也变为二倍
	            newThr = oldThr << 1; // double threshold
	    }
	    else if (oldThr > 0) // initial capacity was placed in threshold
	        newCap = oldThr;
	    // threshold为0，则使用默认值
	    else {               // zero initial threshold signifies using defaults
	        newCap = DEFAULT_INITIAL_CAPACITY;
	        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	    }
	    // 如果临界值还为0，则设置临界值
	    if (newThr == 0) {
	        float ft = (float)newCap * loadFactor;
	        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
	                  (int)ft : Integer.MAX_VALUE);
	    }
	    // 更新填充因子
	    threshold = newThr;
	    @SuppressWarnings({"rawtypes","unchecked"})
	    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	    table = newTab;
	    // 调整数组大小之后，需要调整红黑树或者链表的指向
	    if (oldTab != null) {
	        for (int j = 0; j < oldCap; ++j) {
	            Node<K,V> e;
	            if ((e = oldTab[j]) != null) {
	                oldTab[j] = null;
	                if (e.next == null)
	                    newTab[e.hash & (newCap - 1)] = e;
	                // 红黑树调整
	                else if (e instanceof TreeNode)
	                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
	                else { // preserve order
	                	// 链表调整 分成高high，低low2种
	                    Node<K,V> loHead = null, loTail = null;
	                    Node<K,V> hiHead = null, hiTail = null;
	                    Node<K,V> next;
	                    do {
	                        next = e.next;
			//oldCap是2的幂次方，2进制中只有一个是1其他都是0，和hash值取&后
			//只有一个位是0或者1了，其他为都是0，所以新数组的下标就只有2种情况，高1，低0
			//采用尾插法的方法依次插入到新数组的node里(老数组采用尾插法，扩容后的新数组也采用尾插法
			这样可以避免链表引用冲突)
	                        if ((e.hash & oldCap) == 0) {
	                            if (loTail == null)
	                                loHead = e;
	                            else
	                                loTail.next = e;
	                            loTail = e;
	                        }
	                        else {
	                            if (hiTail == null)
	                                hiHead = e;
	                            else
	                                hiTail.next = e;
	                            hiTail = e;
	                        }
	                    } while ((e = next) != null);
	                    if (loTail != null) {
	                        loTail.next = null;
	                        newTab[j] = loHead;
	                    }
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
	
   红黑树在转移的时候分成高低2种是因为，扩容后index的位置有2种可能，二进制&的原因

   node扩容里loTail，hiTail分别放到2个地方，高，低的意思

   红黑树扩容时候的转移：

   统计低位个数，统计高位的个数，如果高低位的数量小于等于6了，就需要转成node

   如果有低位，没有高位的话直接移动整棵树就行，如果有低位，也有高位，则低位树化，然后高位树化
	
