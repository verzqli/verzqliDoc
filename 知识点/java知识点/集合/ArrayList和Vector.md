## ArrayList

> ArrayList 是 java 集合框架中比较常用的数据结构了。继承自 AbstractList，实现了 List 接口。底层基于数组实现容量大小动态变化。允许 null 的存在。同时还实现了 RandomAccess、Cloneable、Serializable 接口，所以ArrayList 是支持快速访问、复制、序列化的

### 特点

- 【支持类型】：只能装入引用对象（基本类型要转换为封装类）
- 【线程是否安全】：线程不安全
- 【底层数据结构】：底层由数组实现（顺序表），因为由顺序表实现，所以会具备顺序表的特点，如：需要声明长度、超出长度时需要进行扩容、不适合频繁的移动删除元素、检索元素快；
- 【存储数据】：有序容器，即存放元素的顺序与添加顺序相同，允许添加相同元素，包括 null

### 属性

```java
 	//默认初始化大小为10
    private static final int DEFAULT_CAPACITY = 10;

	//初始化时默认为一个空数组，在传递过程中避免没数据时为null
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用于默认大小的空实例的共享空数组实例。我们
	 * 将其与空的元素数据区分开，方便知道添加第一个元素时要扩容多少。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 存储ArrayList元素的数组缓冲区。ArrayList的容量是这个数组缓冲区的长度
     */
    // Android-note: Also accessed from java.util.Collections
    transient Object[] elementData; // non-private to simplify nested class access
    private int size;
	//数组最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

这里注意一点的是``EMPTY_ELEMENTDATA``和``DEFAULTCAPACITY_EMPTY_ELEMENTDATA``

当你使用无参构造函数创建``ArrayList``时，``elementData``等于``DEFAULTCAPACITY_EMPTY_ELEMENTDATA``，且在第一次``add``的时候会自动扩容到默认的10大小。

```java
  if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
       minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
  }
```

当创建的时候传入了一个``Collection<? extends E> c``或者一个初始化大小``initialCapacity``时，``elementData``等于``EMPTY_ELEMENTDATA``

#### 方法

#### add

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
    	return true;
}   
public void add(int index, E element) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        ensureCapacityInternal(size + 1);  // Increments modCount!!
    	//从原数组的index位置拷贝到目标数组的index位置，长度为length
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

增添方法很简单，先判断是否需要扩容，如果是尾部添加，则直接在最后加入即可。

如果是下标添加，先判断有没有越界，再判断是否需要扩容，然后把index之后所有的数据通过拷贝集体往后移动一格，再把index下标的数据赋值。

#### remove

```java
    public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }   
   public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    protected void removeRange(int fromIndex, int toIndex) {
        // Android-changed: Throw an IOOBE if toIndex < fromIndex as documented.
        // All the other cases (negative indices, or indices greater than the size
        // will be thrown by System#arrayCopy.
        if (toIndex < fromIndex) {
            throw new IndexOutOfBoundsException("toIndex < fromIndex");
        }

        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
    
```

``Arraylist``有三种删除的方式，通过下标删除单个或者多个，通过``object``删除

可以看到删除的本质最后还是`` System.arraycopy``,应该说增删的本质就是扩容和数组拷贝。

这里注意到有一个``  modCount++``,增加过程中也有这个，每一次对数组的增删操作都会让``  modCount``自增，这个属性目的是为了检验迭代操作时的正确性。

#### 扩容

```java
  	//minCapacity就是size+1，这里如果elementData是默认构造函数创建的，则扩容到默认大小10
	private void ensureCapacityInternal(int minCapacity就是size+1，这里如果) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }    
	private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //新容量为原容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果扩容了之后还是不够，那么直接让newCapacity等于当前的添加数据后的总大小，等下一次添加再扩容
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
	//这里的MAX_ARRAY_SIZE默认为Integer.MAX_VALUE-8
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

这里可以看到增加时的``modCount++``在这里，扩容默认为原数组容量的1.5倍，但是如果扩容后还是不够大，那么数组大小直接等于添加数据后的大小，等待下一次扩容。

这里的``MAX_ARRAY_SIZE``因为一些vm会在头部 保留一些``header words``，所以会减去8，但是如果强行大于它的话，那么会返回`` Integer.MAX_VALUE``.

#### 迭代器

我们可以在Arraylist中使用迭代器来对数据进行操作

```java
       Iterator iterator =list.iterator();
       	while (iterator.hasNext()){
          System.out.println( iterator.next());
       }
```

以及增强for，增强for也是用迭代器生成的

```java
 for (String s : list) {
     System.out.println(s);
 }
```

迭代器的接口定义

```java
public interface Iterator<E> {
​
    boolean hasNext();
​
    E next();
​
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
​
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

可以看得到迭代器中只定义了四个方法， ``hasNext() ``判断是否有下个元素，`` Next() ``获取下个元素， remove() 删除元素，以及`` forEachRemaining() ``方法对每个剩余元素都执行给定操作，直到所有元素都被处理或动作引发异常。

#### ArrayList中的迭代器

```java
  private class Itr implements Iterator<E> {
        // Android-changed: Add "limit" field to detect end of iteration.
        // The "limit" of this iterator. This is the size of the list at the time the
        // iterator was created. Adding & removing elements will invalidate the iteration
        // anyway (and cause next() to throw) so saving this value will guarantee that the
        // value of hasNext() remains stable and won't flap between true and false when elements
        // are added and removed from the list.
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < limit;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            //modCount排上用场了，expectedModCount初始化时等于modCount
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            //当前的游标赋给i
            int i = cursor;
            //判断是否越界
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                //最后还是调用前面的remove方法，每次删除最后一个
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
                limit--;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;

            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

在迭代过程中对集合做出修改就会判断``modCount != expectedModCount``,如果不相等就抛出异常``ConcurrentModificationException``

modCount是当前集合的版本号，每次对集合进行修改操作都会导致``modCount``加1，``expectedModCount``是这个集合的迭代器的版本号，在迭代器的初始话过程中可以看到`` int expectedModCount = modCount;``

迭代过程中调用``NEXT``和``remove``方法时都会判断这两者是否相等，如果不等就抛出异常，所以在迭代过程中无法对集合进行任何修改的操作，而迭代的``remove``方法在删除后都会重新赋值``expectedModCount = modCount;``。来同步集合和集合迭代器的版本号，因此迭代器的``remove``方法可以堆积和进行删除操作。

增强for循环是迭代器实现的，因此增强for循环和其同理。在增强for循环中修改元素一样会抛出异常。

#### subList

subList是List接口中定义的一个方法，该方法主要用于返回一个集合中的一段、可以理解为截取一个集合中的部分元素，他的返回值也是一个List。

```java
    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList(this, 0, fromIndex, toIndex);
    }
    private class SubList extends AbstractList<E> implements RandomAccess {
        private final AbstractList<E> parent;
        private final int parentOffset;
        private final int offset;
        int size;
        ...
        public void add(int index, E e) {
            if (index < 0 || index > this.size)
                throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
            //调用前会判断两者是否相同
            if (ArrayList.this.modCount != this.modCount)
                throw new ConcurrentModificationException();
            parent.add(parentOffset + index, e);
            //执行完操作后，同步modCont值
            this.modCount = parent.modCount;
            this.size++;
        }

        public E remove(int index) {
            if (index < 0 || index >= this.size)
                throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
            if (ArrayList.this.modCount != this.modCount)
                throw new ConcurrentModificationException();
            E result = parent.remove(parentOffset + index);
            this.modCount = parent.modCount;
            this.size--;
            return result;
        }
```

这里的SubList是ArrayList的一个内部类，这个类中单独定义了set、get、size、add、remove等方法。

当我们调用subList方法的时候，会通过调用SubList的构造函数创建一个SubList

也就是说，SubList并没有重新创建一个List，而是直接引用了原有的List（返回了父类的视图），只是指定了一下他要使用的元素的范围而已（从fromIndex（包含），到toIndex（不包含））。

所以，为什么不能讲subList方法得到的集合直接转换成ArrayList呢？因为SubList只是ArrayList的内部类，他们之间并没有继承关系，故无法直接进行强制类型转换。
另外，视图和原List的修改还需要注意几点，尤其是他们之间的相互影响：

- 对父(sourceList)子(subList)List做的非结构性修改（例如set修改某个下标的值），都会影响到彼此。

- 对子subList做结构性修改，操作同样会反映到父List上。

- 对父sourceList做结构性修改（add remove），会抛出异常ConcurrentModificationException。

那么如何对subList作出修改，又不想动原list。那么可以创建subList的一个拷贝

```java
subList = Lists.newArrayList(subList);
```

这一点在《阿里巴巴Java开发手册中也提到》

> 3.[强制]在sublist场景中，高度注意对原集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生ConcurrentModificationException异常。

#### clone

clone分为**浅克隆和深克隆**

- 浅克隆，即很表层的克隆，如果我们要克隆对象，只克隆它自身以及它所包含的所有**对象的引用地址**

- 深克隆，克隆除自身对象以外的所有对象，包括自身所包含的**所有对象实例、**

那其实Object的clone()方法，提供的是一种浅克隆的机制，如果想要实现对对象的深克隆，在不引入第三方jar包的情况下，可以使用两种办法：

- 先对对象进行序列化，紧接着马上反序列化出

- 先调用super.clone()方法克隆出一个新对象来，然后在子类的clone()方法中手动给克隆出来的非基本数据类型（引用类型）赋值，比如ArrayList的clone()方法：

```java
    public Object clone() {
        try {
            //1、第一句，浅拷贝出一个ArrayList，此时对象的引用地址还是原来的
            ArrayList<?> v = (ArrayList<?>) super.clone();
             //2、第二句，除了自身外，把Object[] elementData里面的对象全部拷贝一次，做了一次深拷贝(具体的原理需要深入的研究copyof方法的源码
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

### 总结

- 默认容量大小为10，无参构造默认空数组为``DEFAULTCAPACITY_EMPTY_ELEMENTDATA``，添加数据后自动扩容到默认大小10，若传入大小或者列表为0 ，默认空数组为``EMPTY_ELEMENTDATA``，不会自动扩容
- 通过modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。
- subList返回来并不是ArrayList,它只是一个内部类，对原集合做任何结构性改变的操作都会抛出异常。
- clone方法是深拷贝，克隆体修改值不会影响原集合



### 扩展： Vector

vector和ArrayList所有代码几乎都相同，只不过他通过给每个方法加锁的方式实现了线程安全。

另外vector的扩容机制是加倍，而且可以自己设置增长因子。如果不指定增长因子，那么就给原数组大小*2.

```java
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
```

