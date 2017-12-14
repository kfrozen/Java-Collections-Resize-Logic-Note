## Java中集合的扩容策略及实现

从源码角度分析Java中常用集合类的扩容机制

从这一篇开始，会陆续通过笔记来整理和记录之前看过的各种Java集合相关的知识点，主要包括List和Map。今天这一篇主要整理一下**集合扩容**相关的知识，涉及到的集合框架有：HashMap，ArrayMap，SparseArray，ArrayList，Vector。下面先从ArrayList开始。

- **ArrayList**

	ArrayList是以数组实现的一个集合类，在ArrayList的源码中可以看到，所有元素都是被储存在elementData这个全局的数组变量中，而所谓的扩容也是针对这个数组对象进行操作。具体来说，当一个添加元素的动作，即add或addAll被执行时，都会先调用ensureCapacityInternal(...)方法进行容量预检，如果当前elementData数组的容量不足以完成本次添加操作便会进行自动扩容。该方法代码如下：
	
	```
		//这是一个私有方法，ArrayList提供了另一个public的扩容方法ensureCapacity以满足外界手动扩容的需求
		//其实质也是调用了本方法，在此不做累述
		//minCapacity是指本次添加操作后所需要的数组容量，即elementData.length + newSize
		private void ensureCapacityInternal(int minCapacity) {
				//这个if判断如果成立，则代表当前ArrayList为空，而本次添加操作是第一次添加。
				//此时需要将ArrayList的容量扩充至DEFAULT_CAPACITY=10和本次添加操作所要求的minCapacity二者中的较大者。
        		if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            		minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        		}

        		ensureExplicitCapacity(minCapacity);
    	}

    	private void ensureExplicitCapacity(int minCapacity) {
        	//操作计数器+1
        	modCount++;

        	// overflow-conscious code
        	// 确保需要进行扩容，即当前数组的容量小于本次添加操作所要求的新的总容量
        	if (minCapacity - elementData.length > 0)
            	grow(minCapacity);
    	}
	```
	在ensureCapacityInternal方法中，一开始会对本次添加操作是否为第一次添加进行判断。从源码：***private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};*** 中可以得知，DEFAULTCAPACITY_EMPTY_ELEMENTDATA变量指代的是一个空的对象数组，这也是通过无参构造函数new出来的ArrayList对象的初始状态，也即 elementData = {}；这种情况下的第一次扩容，如果所要求的新容量小于10，则会直接扩容至10。
	
	下面继续看到grow方法，这个方法是ArrayList扩容操作的实现
	
	```
		private void grow(int minCapacity) {
        	// overflow-conscious code
        	//记录当前elementData数组的容量
        	int oldCapacity = elementData.length;
        	//新的容量暂定为当前容量的1.5倍
        	int newCapacity = oldCapacity + (oldCapacity >> 1);
        	//如果1.5倍容量仍不满足minCapacity所要求的，则直接将容量定为minCapacity
        	if (newCapacity - minCapacity < 0)
            	newCapacity = minCapacity;
          	//如果新的容量超过了Integer.MAX_VALUE - 8，则做最大化处理
        	if (newCapacity - MAX_ARRAY_SIZE > 0)
            	newCapacity = hugeCapacity(minCapacity);
        	// minCapacity is usually close to size, so this is a win:
        	//将当前数组copy到新的数组中
        	elementData = Arrays.copyOf(elementData, newCapacity);
    	}

    	private static int hugeCapacity(int minCapacity) {
        	if (minCapacity < 0) // overflow
            	throw new OutOfMemoryError();
        	return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    	}
	``` 
	最后的扩容操作，Arrays.copyOf方法会新创建一个大小为newCapacity的数组，然后通过System.arraycopy方法将之前elementData数组中的元素复制到新数组中(起点的index为0)，并将新数组返回给变量elementData完成扩容。

- **Vector**

	Vector和ArrayList其实身出同门，都是AbstractList的子类并且都实现了List<E>接口，所提供的功能也基本相同，最大的不同在于Vector所有的公有API都是加锁的，也即Vector是线程安全的。但这也正是它不受欢迎的原因之一，太重了。。。言归正传，我们来关心一下Vector的扩容操作。其实基本跟ArrayList相同，方法如下：
	
	```
		private void ensureCapacityHelper(int minCapacity) {
        	// overflow-conscious code
        	//当前容量不够，扩之
        	if (minCapacity - elementData.length > 0)
            	grow(minCapacity);
    	}
	```
	可以看到，基本只是方法名不一样而已，再来看看Vector的grow函数，这里还真有些不同：
	
	```
		private void grow(int minCapacity) {
        	// overflow-conscious code
        	int oldCapacity = elementData.length;
        	
        	//不同之处在这里，MARK
        	int newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity);
        	
        	if (newCapacity - minCapacity < 0)
            	newCapacity = minCapacity;
       	if (newCapacity - MAX_ARRAY_SIZE > 0)
            	newCapacity = hugeCapacity(minCapacity);
        	elementData = Arrays.copyOf(elementData, newCapacity);
    	}
	```
	上面代码块中MARK出的那一句便是不同之处，可以看出Vector的扩容是先判断有没有大于0的capacityIncrement，该变量是通过：
	
	```
		public Vector(int initialCapacity, int capacityIncrement) {
        	super();
        	if (initialCapacity < 0)
            	throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        	this.elementData = new Object[initialCapacity];
        	this.capacityIncrement = capacityIncrement;
    	}
	```
	这个带参构造函数传入的，如果不调用这个构造函数，则capacityIncrement为0。**在capacityIncrement不为0的情况下，则每次扩容都暂定扩大capacityIncrement个，反之扩容oldCapacity个，即当前容量的一倍。**后续的扩容操作和ArrayList一致，也是通过Arrays.copyOf方法来完成，这里不再重复分析。
	
- **HashMap**

	下面来看另一个熟人--HashMap。不同于上面两个List的实现类，HashMap是一个采用哈希表实现的键值对集合，继承自AbstractMap，实现了Map接口并使用拉链法解决Hash冲突，其内部储存的元素并不是在连续内存地址的，并且是无序的。此处我们只关心其扩容操作的逻辑和实现，先说一下，由于要重新创建数组，rehash，重新分配元素位置等，HashMap扩容的开销要比List大很多。下面介绍几个和扩容相关的成员变量：
	
	```
		//哈希表中的数组，JDK 1.8之前存放各个链表的表头。1.8中由于引入了红黑树，则也有可能存的是树的根
		transient Node<K,V>[] table;
	
		//默认初始容量：16，必须是2的整数次方。这样规定是因为在通过key来确定元素在table中的index时
		//所用的算法为:index = (n - 1) & hash，其中n即为table容量。保证n是2的整数次方就能保证n-1的低位均为1，
		//这样便能保留hash(key)得到的hash值的所有低位，从而保证得到的index在n范围内分布均匀，因为hash算法的结果就是均匀的
		static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
		
		//默认加载因子为: 0.75，这是在时间、空间两方面均衡考虑下的结果。
		//这个值过大会导致发生冲突的几率增加，容易形成长链表，降低查找效率；太小则会导致频繁的扩容，降低整体性能。
		static final float DEFAULT_LOAD_FACTOR = 0.75f;
		
		//阈值，下次需要扩容时的值，等于 容量*加载因子
		int threshold;
		
		//最大容量： 2^30次方
		static final int MAXIMUM_CAPACITY = 1 << 30;
		
		//树化阈值。JDK 1.8后HashMap对冲突处理做了优化，引入了红黑树。
		//当桶中元素个数大于TREEIFY_THRESHOLD时，就需要用红黑树代替链表，以提高操作效率。此值必须大于2，并建议大于8
		static final int TREEIFY_THRESHOLD = 8;
		
		//非树化阈值。在进行扩容操作时，桶中的元素可能会减少，这很好理解，因为在JDK1.7中，
		//每一个元素的位置需要通过key.hash和新的数组长度取模来重新计算，而1.8中则会直接将其分为两部分。
		//并且在1.8中，对于已经是树形的桶，会做一个split操作(具体实现下面会说)，在此过程中，
		//若剩下的树元素个数少于UNTREEIFY_THRESHOLD，则需要将其非树化，重新变回链表结构。
		//此值应小于TREEIFY_THRESHOLD，且规定最大值为6
		static final int UNTREEIFY_THRESHOLD = 6;
		
	```
	好了，相关变量介绍完了，接下来开始分析HashMap的扩容函数resize，一个长得很讨厌的方法:
	
	```
	final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        //记录当前数组长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //记录当前扩容阈值
        int oldThr = threshold;
        int newCap, newThr = 0;
        //下面一长串if-else是为了确定newCap和newThr，即新的容量和扩容阈值
        if (oldCap > 0) {
        	//oldCap不为0，已被初始化过
            if (oldCap >= MAXIMUM_CAPACITY) {
            	//当前已经是最大容量，不允许再扩容，返回当前哈希表
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //先将oldCap翻倍，如果得到的值小于最大容量，并且oldCap不小于默认初始值，则将扩容阈值也翻倍，结束
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) 
        	// initial capacity was placed in threshold
        	// 若构造函数中有传入initialCapacity，则会暂存在oldThr=threshold变量中。
        	// 然后在第一次put操作导致的resize方法中被赋给newCap，这样做的目的应该是避免污染oldCap从而影响上面那个if的判断
        	// 从这里也可以看出HashMap对于所需内存的申请是被延迟到第一次put操作时进行的，而非在构造函数中。
            newCap = oldThr;
        else {               
        	// zero initial threshold signifies using defaults
        	// Map没有被初始化，用默认值初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
        	//若经过计算后，新阈值为0，则赋值为新容量和扩容因子的乘积(需考虑边界条件)
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        
        /******-----分割线，至此新的容量和扩容因子已确定------*************/
        
        @SuppressWarnings({"rawtypes","unchecked"})
        //创建一个大小为newCap的Node<K,V>数组，并赋值给table变量
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
        	//遍历扩容前的数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                	//将原数组中第j个元素赋给e，并将原数组第j位置置空
                    oldTab[j] = null;
                    if (e.next == null)
                    	//该元素没有后续节点，即该位置未发生过hash冲突。则直接将该元素的hash值与新数组的长度取模得到新位置并放入
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                    	//JDK1.8中，如果该元素是一个树节点，说明该位置存放的是一颗红黑树，则需要对该树进行分解操作
                    	//具体实现后面会讨论，这里split的结果就是分为两棵树(这里必要时要进行非树化操作)并分别放在新数组的高段和低段
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                    	//剩下这种情况就是该位置存放的是一个链表，需要说明的是在JDK1.7和1.8中这里有着不同的实现，下面分别讨论
                    	
                    	/******--- JDK 1.7版本 starts ---*******/
                    	//遍历链表
                    	while(null != e) {
                    		//将原链表中e的下一个元素暂存到next变量中
                			HashMapEntry<K,V> next = e.next;
                			//算出在新数组的index=i，indexFor其实就是e.hash & (newCapacity - 1)
                			int i = indexFor(e.hash, newCapacity);
                			//改变e.next的指向，将新数组该位置上原先的内容(一个链表，元素或是null)挂在e的身后，使e成为这个链表的表头
                			e.next = newTable[i];
                			//将这个e位表头的新链表放回到index为i的位置中
                			newTable[i] = e;
                			//将之前暂存的原链表中的下一个元素赋给e，继续遍历原链表
                			e = next;
            			}
                    	/******--- JDK 1.7版本 ends ---*******/
                    	
                    	/******--- JDK 1.8版本 starts ---*******/
                    	//在1.8的实现中，新数组被分成了高低两个段，而原链表也会被分成两个子链表，分别放入新数组的高段和低段中
                    	//loHead和loTail用于生成将被放入新数组低段的子链表
                        Node<K,V> loHead = null, loTail = null;
                        //hiHead和hiTail则用于生成将被放入新数组高段的子链表
                        Node<K,V> hiHead = null, hiTail = null;
                        //跟1.7中一样，next用于暂存原链表中e的下一个元素
                        Node<K,V> next;
                        //开始遍历原链表
                        do {
                            next = e.next;
                            //用if中的方法确定e是该去新数组的高段还是低段
                            if ((e.hash & oldCap) == 0) {
                            	//将e加到将被放入低段的子链表的尾部
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                            	//将e加到将被放入高段的子链表的尾部
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                       
                        if (loTail != null) {
                        	//将loHead指向的子链表放入新数组中index=j的位置
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                        	////将hiHead指向的子链表放入新数组中index=j+oldCap的位置
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                        /******--- JDK 1.8版本 ends ---*******/
                    }
                }
            }
        }
        return newTab;
    }
	```
	可以看到JDK1.8对resize方法进行了彻底的改造，引入红黑树结合之前的链表势必会提高在发生hash冲突时的操作效率(红黑树能保证在最坏情况下插入，删除，查找的时间复杂度都为O(logN))。
	
	此外最大的改动便是在扩容的时候对链表或树的处理，在1.7时代，链表中的每一个元素都会被重新计算在新数组中的index，具体方法仍旧是e.hash对新数组长度做取模操作；而在1.8时代，这个链表或树会被分为两部分，我们暂且称其为A和B，若元素的hash值按位与扩容前数组的长度得到的结果为0（其实就是判断hash的某一位是1还是0，由于hash值均匀分布的特性，这个分裂基本可以认为是均匀的），则将其接入A，反之接入B。最后保持A的位置不变，即在新数组中仍位于原先的index=j处，而B则去到j+oldCap处。
	
	**其实对于这个改动带来的好处我理解的不是特别透彻，因为整个过程并没有减少计算的次数。目前看到的好处是可以避免扩容重定向过程中发生哈希冲突（因为是扩容一倍，所以一个萝卜一个坑，不会有冲突），并且不会将链表中的元素倒置（考虑极端情况，就一条链表，1.7的方法每次都会将元素插到表头）。这里还是得求教大家，欢迎讨论~**
	
	回到resize方法，上面还留了一个尾巴，就是当桶中是树形结构时的split方法，下面就来看源码：
	
	```
		 /**
         * Splits nodes in a tree bin into lower and upper tree bins,
         * or untreeifies if now too small. Called only from resize;
         * see above discussion about split bits and indices.
         *
         * @param map the map
         * @param tab the table for recording bin heads
         * @param index the index of the table being split
         * @param bit the bit of hash to split on
         */
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        	TreeNode节点继承自LinkedHashMapEntry，在链表节点的基础上扩充了树节点的功能，譬如left，right，parent
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            // 将树分为两部分这里的做法和链表结构时是相似的
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            //遍历整棵树，这里说明一下，由于这棵树是由链表treeify生成的，其next指针依旧存在并指向之前链表中的后继节点，
            //因此遍历时依然可以按照遍历链表的方式来进行
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
            	//暂存next
                next = (TreeNode<K,V>)e.next;
                //将e从链表中切断
                e.next = null;
                //与对链表的处理相同，若e.hash按位与bit=oldCap结果为0，则接到低段组的尾部
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                //否则接到高段组的尾部
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

			//若低段不为空
            if (loHead != null) {
            	//如果低段子树的元素个数小于非树化阈值，则将该树进行非树化，还原为链表
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                	//否则的话将低段子树按照原本在旧数组中的index放入新数组中
                    tab[index] = loHead;
                    //对低段子树进行树化调整。这里有一个优化，如果发现高段子树为空，则说明之前树中的所有元素都被放到了低段子树中，
                    //也即这已经是一棵完整的，调整好了的红黑树，不需要再进行树化调整
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
            	//与低段子树同样的逻辑，放入新数组的位置为旧数组的index+oldCap
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    //这里同样的，如果低段子树为空，说明高段这棵树已经是一棵完整的红黑树，无需调整
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
	```
	**注：由于篇幅所限，红黑树的treeify树化操作和untreeify非树化操作将在另一篇关于红黑树的文章中单独进行说明，在此大家只需理解treeify做的事情是将一个链表中的TreeNode节点，按照二叉查找树的结构连接其left，right和parent指针，并根据红黑树的规则进行调整，同时也可以对一个非红黑树的树结构进行调整；而untreeify反之，是将一个红黑树的TreeNode节点还原为HashMap.Node节点，并将其首尾相连还原为一个链表结构。**
	
	至此，HashMap的扩容逻辑和实现就分析完成了，可以看到1.7中的逻辑和做法还比较简单粗暴，而到了1.8中由于红黑树的引入，整体变得精巧了许多，整体HashMap的操作性能也有了大的提升。但即便如此，HashMap的扩容依旧是一个很贵的操作，这就要求我们在初始化HashMap的时候根据自己的业务场景设置尽可能合适的初始容量，以降低扩容发生的几率。例如我需要一个容纳96个元素的map，那么只要我把capacity初始值设置为128，那么就不会经历16到32到64再到128的三次扩容，这样来说是节省内存和运算成本的。当然如果需要容纳97个元素的话，因为超过了capacity值的3/4，所以就需要设置为256了，否则也会经历一次扩容。
	
- **ArrayMap**

	ArrayMap是一个<key,value>映射的数据结构，它设计上更多的是考虑内存的优化，内部是使用两个数组进行数据存储，一个数组记录key的hash值，另外一个数组记录Value值。它会对key使用二分法进行从小到大排序，在添加、删除、查找数据的时候都是先使用二分查找法得到相应的index，然后通过index来进行添加、查找、删除等操作。相比于HashMap，它更适合在数据量不是很大的情况下(千级)使用，有点用时间换空间的意思，如果在数据量比较大的情况下，那么它的性能将退化至少50%。
	
	下面还是先来看看跟ArrayMap扩容相关的成员变量：
	
	```
			//最小扩容容量，即一次扩容最少扩大4个
    		private static final int BASE_SIZE = 4;
    
			//第一个数组，存放所有key的hash值，这个数组的容量就是ArrayMap当前的容量
			int[] mHashes;
			
			//第二个数组，容量是mHashes的两倍，元素的key和value被相邻存放，即一个元素占两格
    		Object[] mArray;
    		
    		//ArrayMap中当前元素的个数，也即mHashes数组中元素的个数
    		int mSize;
    		
    		//小号缓存数组，数组长度为8，其中index=0位置存放的是指向下一个缓存数组index=0位置的指针，
    		//即该位置存储的其实是一个链表，链表元素为一个一个的mBaseCache数组，其通过index=0位置相连
    		//index=1位置存放的是一个长度为4的数组，供mHashes使用
			static Object[] mBaseCache;
			
			//小号缓存数组中index=0的位置存放的链表的长度
    		static int mBaseCacheSize;
    		
    		//大号缓存数组，数组长度为16，其中index=0位置存放的是指向下一个缓存数组index=0位置的指针，
    		//即该位置存储的其实是一个链表，链表元素为一个一个的mTwiceBaseCache数组，其通过index=0位置相连
    		//index=1位置存放的是一个长度为8的数组，供mHashes使用
    		static Object[] mTwiceBaseCache;
    		
    		//大号缓存数组中index=0的位置存放的链表的长度
    		static int mTwiceBaseCacheSize;
    		
    		//缓存数组中的链表的最大长度，不能超过10
    		private static final int CACHE_SIZE = 10;
	```
	相关成员变量就是这些，先提一句，ArrayMap不同于HashMap，如果你在构造函数中指定了capacity，则构造函数会直接为mHashes和mArray两个数组申请内存，并不会延迟到第一次put。如果不指定capacity，则两个数组会被分别赋值为EmptyArray.INT和EmptyArray.OBJECT，即对于类型的空数组。下面我们来看扩容的实现：
	
	```
		   /**
    		* Ensure the array map can hold at least <var>minimumCapacity</var>
     		* items.
     		*/
     		//这个方法在putAll中会被调用，传入的参数是mSize+array.size()，即当前个数与新put进来的集合个数的和。
     		//这个方法并没有在put中被调用，但核心方法都是allocArrays(capacity)，唯一的差别在于这个capacity的计算方法，
     		//所以此处我们先用它来分析，至于计算capacity的区别后面会提到。
     		//minimumCapacity这个输入参数表示新的操作需要这个容量来支撑，也即新的最小容量
    		public void ensureCapacity(int minimumCapacity) {
    			//暂存扩容前的元素个数至osize
        		final int osize = mSize;
        		//若当前ArrayMap的容量小于所需的最小容量，则进行扩容
        		if (mHashes.length < minimumCapacity) {
        			//暂存扩容前的mHashes数组至ohashes
            		final int[] ohashes = mHashes;
            		//暂存扩容前的mArray数组至oarray
            		final Object[] oarray = mArray;
            		//申请内存，核心操作，后面单独分析
            		allocArrays(minimumCapacity);
            		//若ArrayMap中有元素
            		if (mSize > 0) {
            			//这里的mHashes是经过扩容后的空数组，这一句是将扩容前暂存的ohashes中的内容copy到新的mHashes中
                		System.arraycopy(ohashes, 0, mHashes, 0, osize);
                		//同理，这里的mArray是经过扩容后的空数组，这里将扩容前暂存的oarray中的内容copy到新的mArray中
                		System.arraycopy(oarray, 0, mArray, 0, osize<<1);
            		}
            		//善后工作，该缓存的缓存，该置空的置空
            		freeArrays(ohashes, oarray, osize);
        		}
        		if (CONCURRENT_MODIFICATION_EXCEPTIONS && mSize != osize) {
            		throw new ConcurrentModificationException();
        		}
    		}
	```
	下面继续看真正干活儿的**allocArrays**方法，上面说到的putAll方法会通过ensureCapacity来调用到这个方法，传入的size为mSize+array.size()。除此之外，构造函数中根据指定的capacity初始化数组的操作也是通过这个函数完成的，size自然就是指定的capacity。但是最常用到的还是put方法，即添加单个元素，put中调用allocArrays(size)时传入的size是这样算出来的：
	
	```
		final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                    : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);
	```
	翻译一下就是先判断mSize值是否大于等于8，如果是则n=mSize*1.5，如果否，则判断是否大于等于4，是则n=8个，否则n=4个。
	
	```
		private void allocArrays(final int size) {
        	if (mHashes == EMPTY_IMMUTABLE_INTS) {
            	throw new UnsupportedOperationException("ArrayMap is immutable");
        	}
        	if (size == (BASE_SIZE*2)) {
        		//线程安全，并且上的是类锁
            	synchronized (ArrayMap.class) {
            		//若要求的size是8，且大号缓存数组不为空，则从大号缓存数组中取缓存
                	if (mTwiceBaseCache != null) {
                		//将当前大号缓存数组赋给array，其容量为16，符合mArray的要求
                    	final Object[] array = mTwiceBaseCache;
                    	mArray = array;
                    	//index=0的位置存放的是指向下一个大号缓存数组的指针，取出赋给mTwiceBaseCache变量，即第一层缓存被mArray取走了
                    	mTwiceBaseCache = (Object[])array[0];
                    	//第一层缓存的index=1的位置存放的是长度为8的数组，赋给mHashes
                    	mHashes = (int[])array[1];
                    	//将已经取出的第一层缓存数组的前两位都清空，这样一来便得到了长度分别为16和8的空的mArray和mHashes数组，好精巧的设计！
                    	array[0] = array[1] = null;
                    	//大号缓存被取走了一层，链表长度减一
                    	mTwiceBaseCacheSize--;
                    	if (DEBUG) Log.d(TAG, "Retrieving 2x cache " + mHashes
                            	+ " now have " + mTwiceBaseCacheSize + " entries");
                    	return;
                	}
            	}
        	} else if (size == BASE_SIZE) {
        		//这边size=4，与上同理，只是操作的缓存对象变成了小号缓存数组
            	synchronized (ArrayMap.class) {
                	if (mBaseCache != null) {
                    	final Object[] array = mBaseCache;
                    	mArray = array;
                    	mBaseCache = (Object[])array[0];
                    	mHashes = (int[])array[1];
                    	array[0] = array[1] = null;
                    	mBaseCacheSize--;
                    	if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                            	+ " now have " + mBaseCacheSize + " entries");
                    	return;
                	}
            	}
        	}

			//如果要求的size不是4或8，则不满足使用缓存的条件，直接按照要求的size新创建两个数组
        	mHashes = new int[size];
        	mArray = new Object[size<<1];
    	}
	```
	上面缓存的逻辑有点绕，还有一个缓存相关的地方需要分析一下，就是freeArrays这个方法，不要被它的名字迷惑了，它不光是做了回收内存，上面两个缓存数组的添加就是在这里完成的，下面来看一下：
	
	```
		//调用时是这个样子的，三个参数分别为扩容前暂存的mHashes,mArrays和mSize
		freeArrays(ohashes, oarray, osize);
		
		private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
        	if (hashes.length == (BASE_SIZE*2)) {
            	synchronized (ArrayMap.class) {
            		//若需要回收的容量为8，并且大号缓存数组的链表长度还没到10
                	if (mTwiceBaseCacheSize < CACHE_SIZE) {
                		//直接将传入的array数组（显然，此处长度为16）作为新一层的缓存，index=0的位置指向现有的缓存数组
                    	array[0] = mTwiceBaseCache;
                    	//index=1的位置指向本次传入的hashes数组，也即将这个hashes缓存在此
                    	array[1] = hashes;
                    	//将后面14个位置都置空，因为已经无用，避免内存泄漏
                    	for (int i=(size<<1)-1; i>=2; i--) {
                        	array[i] = null; //for gc
                    	}
                    	//将这层新做好的缓存数组赋值给mTwiceBaseCache变量，下次allocArray中就能通过mTwiceBaseCache变量取到这一层缓存。这里缓存层也是后进先出，类似于栈。
                    	mTwiceBaseCache = array;
                    	//大号缓存数组的链表长度加1
                    	mTwiceBaseCacheSize++;
                    	if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                            + " now have " + mTwiceBaseCacheSize + " entries");
                	}
            	}
        	} else if (hashes.length == BASE_SIZE) {
            	synchronized (ArrayMap.class) {
            		//这边size=4，与上同理，只是操作的缓存对象变成了小号缓存数组
                	if (mBaseCacheSize < CACHE_SIZE) {
                    	array[0] = mBaseCache;
                    	array[1] = hashes;
                    	for (int i=(size<<1)-1; i>=2; i--) {
                        	array[i] = null;
                    	}
                    	mBaseCache = array;
                    	mBaseCacheSize++;
                    	if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                            + " now have " + mBaseCacheSize + " entries");
                	}
            	}
        	}
    	}
	```
	到这里ArrayMap的扩容机制就分析完毕了，可以看到ArrayMap内部对于内存的使用进行很大的优化，不仅将每次扩容的容量通过4和8分为三个档位，更是对于4和8的数组采取了缓存的机制，以避免重复创建对象。尤其是缓存结构的设计不可谓不精妙，看完之后只能感叹路还很长，自己何时能设计出如此精妙的结构！
	
	言归正传，其实ArrayMap是在Android SDK中存在，专门供Android使用以在数据量不大的场景下替换HashMap的。通过上面对两者扩容机制的分析不难看出其在这方面的区别：HashMap每次扩容是直接申请双倍于当前容量的内存，而ArrayMap则是根据所需size大小，如果size长度大于8时申请size*1.5个长度，大于4小于8时申请8个，小于4时申请4个。这样一来同等情况下，粒度更细的ArrayMap自然会申请更少的内存空间，同时导致的问题就是扩容频率会大于HashMap。另外由于ArrayMap在查找元素使用的是二分法，当数据量过大时性能会远不如用hash值定位的HashMap。但众所周知内存对于移动设备来说有多么珍贵，因此ArrayMap这种用时间换空间的做法，在数据量较小的时候是非常适合于Android应用的。
	
- **SparseArray**

	最后再来看一下SparseArray，与ArrayMap类似，这也是Android对于HashMap的一种优化和替代方案，不同的是它只能将int作为key的类型（注意是int不是Integer），也就是说它不会对key进行自动装箱，这个是当key为int时SparseArray优于ArrayMap的地方，所以对于数据量不大，且确定key为int类型的场景，可以使用SparseArray代替HashMap或ArrayMap，因为它避免了自动装箱的过程。
	
	SparseArray内部也是通过两个数组来存储数据，一个存key，一个存value。由于key只能是int，所以比较时只需要看是否相等即可，所以结构非常简单。它的扩容是通过GrowingArrayUtils.growSize(int currentsize)方法来完成的，这个工具类位于android.support.v7.content.res包中，其中输入参数是指扩容前当前的数组容量，这个方法非常简单：
	
	```
		public static int growSize(int currentSize) {
			//当前容量不大于4，就扩容到8，否则就扩容一倍
        	return currentSize <= 4 ? 8 : currentSize * 2;
    	}
	```
	
- **最后**

	至此几个常用的集合类的扩容机制就都分析完毕了。后续会继续对这些集合类的查找，添加，删除操作，线程安全性以及HashMap中的红黑树等方面做源码分析，感谢阅读！
	
- **参考**

	http://blog.csdn.net/u011240877/article/details/53351188
	
	http://blog.csdn.net/u011240877/article/details/53358305
	
	http://blog.csdn.net/vansbelove/article/details/52422087

	
