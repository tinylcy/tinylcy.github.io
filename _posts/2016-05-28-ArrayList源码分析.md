---
title: ArrayList源码分析
date: 2016-05-28 12:27:34
tags: [JDK,Java]
toc: true
---

源码版本为`JDK1.7.0_79`

`ArrayList`不是线程安全的，只能应用在单线程环境下。

## ArrayList类定义


从`ArrayList`的类定义可以看出它是支持泛型的，继承自`AbstractList`，`AbstractList` 实现了`List`接口，提供了`List`接口的默认实现。

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>
```

`ArrayList`自身也实现了`List`接口。同时，`ArrayList`实现了`Serializable`接口，因此它支持序列化，能够通过序列化传输。实现了`RandomAccess`接口，支持快速随机访问，实际上就是通过下标进行快速访问，`RandomAccess`是一个标记接口，接口内没有定义任何内容。实现了`Cloneable`接口，能被克隆。
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## ArrayList类属性


`ArrayList`是基于数组实现的，是一个动态数组，其容量能够自动增长。

```java
/**
* 序列版本号
*/
private static final long serialVersionUID = 8683452581122892189L;

/**
* Default initial capacity. 
*/
private static final int DEFAULT_CAPACITY = 10;

/**
* Shared empty array instance used for empty instances.
*/
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
* The array buffer into which the elements of the ArrayList are stored.
* The capacity of the ArrayList is the length of this array buffer. Any
* empty ArrayList with elementData == EMPTY_ELEMENTDATA will be expanded to
* DEFAULT_CAPACITY when the first element is added.
*/
private transient Object[] elementData;

/**
* The size of the ArrayList (the number of elements it contains).
*
* @serial
*/
private int size;
```

数组`elementData`为`ArrayList`存储元素的`Buffer`，`ArrayList`的容量即为该数组的长度。当`ArrayList`为空时，此时`elementData==EMPTY_ELEMENTDATA`，当往`ArrayList`添加一个元素时，`elementData`的长度就被初始化为`10`,也就是`DEFAULT_CAPACITY`的值。

`size`为`elementData`含有的元素的个数。

注意到`elementData`数组被`transient`修饰，因为`ArrayList`实现了`Serializable`接口，这意味着`ArrayList`可以被序列化，也就是说，`ArrayList`的属性可以通过网络传输，或者被存储到磁盘持久化。但是在很多情况下，我们并不希望有些属性被传输或者存储，比如一些敏感信息。这时使用`trainsent`标记属性，就可以使得`ArrayList`在序列化的过程中`elementData`不参与序列化。因此，`elementData`中的数据仅存在于调用者的内存中。

## ArrayList类构造函数


`ArrayList`一共定义了三个构造函数。

```java
/**
* Constructs an empty list with the specified initial capacity.
*/
public ArrayList(int initialCapacity) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                          initialCapacity);
    this.elementData = new Object[initialCapacity];
}

/**
* Constructs an empty list with an initial capacity of ten.
*/
public ArrayList() {
    super();
    this.elementData = EMPTY_ELEMENTDATA;
}

/**
* Constructs a list containing the elements of the specified
* collection, in the order they are returned by the collection's
* iterator.
*/
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
}
```

构造函数`public ArrayList(int initialCapacity){...}`为构造给定大小的`elementData`数组。

构造函数`public ArrayList() {...}`为`ArrayList`的默认构造函数，此时`elementData==EMPTY_ELEMENTDATA`，数组长度为`0`。

构造函数 `public ArrayList(Collection<? extends E> c) {...}`使用一个集合来初始化`ArrayList`。

## ArrayList类核心方法


```java
/**
* Trims the capacity of this <tt>ArrayList</tt> instance to be the
* list's current size.  An application can use this operation to minimize
* the storage of an <tt>ArrayList</tt> instance.
*/
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = Arrays.copyOf(elementData, size);
    }
}
```
`modCount`为`ArrayList`的父类`AbstractList`中定义的属性，其作用是 `the number of times this list has been structurally modified`。

`trimToSize`的作用是将`ArrayList`对象的`capacity`减小到`size`长度，这样可以减少`ArrayList`对象占用的内存。

```java
/**
* Increases the capacity of this <tt>ArrayList</tt> instance, if
* necessary, to ensure that it can hold at least the number of elements
* specified by the minimum capacity argument.
*/
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != EMPTY_ELEMENTDATA)
        // any size if real element table
        ? 0
        // larger than default for empty table. It's already supposed to be
        // at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
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

/**
* The maximum size of array to allocate.
* Some VMs reserve some header words in an array.
* Attempts to allocate larger arrays may result in
* OutOfMemoryError: Requested array size exceeds VM limit
*/
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
* Increases the capacity to ensure that it can hold at least the
* number of elements specified by the minimum capacity argument.
*/
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
`oldCapacity + (oldCapacity >> 1)`确保了一旦扩充容量，那么至少扩充到原来的`1.5`倍。

```java
/**
* Returns the number of elements in this list.
*/
public int size() {
    return size;
}

/**
* Returns <tt>true</tt> if this list contains no elements.
*/
public boolean isEmpty() {
    return size == 0;
}
```
`size`方法和`isEmpty`方法都是根据属性`size`判断。

```java
/**
* Returns <tt>true</tt> if this list contains the specified element.
* More formally, returns <tt>true</tt> if and only if this list contains
* at least one element <tt>e</tt> such that
* <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>.
*/
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}

/**
* Returns the index of the first occurrence of the specified element
* in this list, or -1 if this list does not contain the element.
* More formally, returns the lowest index <tt>i</tt> such that
* <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
* or -1 if there is no such index.
*/
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
`contains`方法中，通过`indexOf`方法遍历`elementData`数组。若待搜索的元素为`null`，那么就返回`elementData`中第一个为`null`的元素的下标。当然如果`elementData`的`size==0`，那么就返回`-1`。若待搜索的元素不为空，那么就返回对应的元素的下标，否则返回`-1`。注意当带搜索的元素不为`null`时，是使用`equals`方法来判断是否相等。

```java
/**
* Returns the index of the last occurrence of the specified element
* in this list, or -1 if this list does not contain the element.
* More formally, returns the highest index <tt>i</tt> such that
* <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
* or -1 if there is no such index.
*/
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
`lastIndexOf`方法与`indexOf`方法十分类似，只是它是从最后一个元素开始向前遍历。

```java
/**
* Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
* elements themselves are not copied.)
*/
public Object clone() {
    try {
        @SuppressWarnings("unchecked")
            ArrayList<E> v = (ArrayList<E>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError();
    }
}
```
`clone`方法首先调用父类的`clone`方法返回一个对象的副本，将这个副本对象的`elementData`数组地址赋值为原来对象的`elementData`数组的地址，并将该副本对象的`modCount`重置为`0`。该方法返回的是`ArrayList`对象的浅副本，即不复制这些元素本身。

```java
/**
* Returns an array containing all of the elements in this list
* in proper sequence (from first to last element).
*
* <p>The returned array will be "safe" in that no references to it are
* maintained by this list.  (In other words, this method must allocate
* a new array).  The caller is thus free to modify the returned array.
*
* <p>This method acts as bridge between array-based and collection-based
* APIs.
*
*/
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

/**
* Returns an array containing all of the elements in this list in proper
* sequence (from first to last element); the runtime type of the returned
* array is that of the specified array.  If the list fits in the
* specified array, it is returned therein.  Otherwise, a new array is
* allocated with the runtime type of the specified array and the size of
* this list.
*
* <p>If the list fits in the specified array with room to spare
* (i.e., the array has more elements than the list), the element in
* the array immediately following the end of the collection is set to
* <tt>null</tt>.  (This is useful in determining the length of the
* list <i>only</i> if the caller knows that the list does not contain
* any null elements.)
*/
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
`toArray`方法返回的是`elementData`数组的副本，而不是`ArrayList`对象内的`elementData`数组本身。

```java
@SuppressWarnings("unchecked")
E elementData(int index) {
    return (E) elementData[index];
}
```
`elementData`方法返回指定下标的元素，`ArrayList`的`get`方法，`set`方法，`add`方法，`remove`方法和`clear`方法都需要调用它。

```java
/**
* Returns the element at the specified position in this list.
*/
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```
`get`方法首先需要校验下标是否越界，若没有，则返回对应下标的元素。

```java
/**
* Replaces the element at the specified position in this list with
* the specified element.
*/
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```
`set`方法将`ArrayList`指定下标的元素设置为指定的值，并将原值作为返回值返回。

```java
/**
* Appends the specified element to the end of this list.
*/
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

/**
* Inserts the specified element at the specified position in this
* list. Shifts the element currently at that position (if any) and
* any subsequent elements to the right (adds one to their indices).
*/
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                    size - index);
    elementData[index] = element;
    size++;
}
```
`add(int index, E element)`方法在指定的位置插入一个元素，首先将`elementData`中从`index`开始的 `size - index` 个元素整体向后移动一位，然后将`element`元素插入到`index`处。

```java
/**
* Removes the element at the specified position in this list.
* Shifts any subsequent elements to the left (subtracts one from their
* indices).
*/
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                        numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```
如果要`remove`的元素为`elementData`的最后一个元素，那么`numMoved`等于`0`，因此不需要做数组拷贝的操作。若要删除的元素位于`elementData`之间，那么需要将`index`后的元素整体向前移动一位，并将`elementData`最后的元素置为`null`值让`GC`回收。

```java
/**
* Removes the first occurrence of the specified element from this list,
* if it is present.  If the list does not contain the element, it is
* unchanged.  More formally, removes the element with the lowest index
* <tt>i</tt> such that
* <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
* (if such an element exists).  Returns <tt>true</tt> if this list
* contained the specified element (or equivalently, if this list
* changed as a result of the call).
*/
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

/*
* Private remove method that skips bounds checking and does not
* return the value removed.
*/
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                        numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```
`fastRemove`和`remove`非常类似，仅仅是移除某个元素。`remove(Object o)`是将`elementData`中的某个对象移除(假设存在)，若传入的要移除的元素为`null`，那么移除`elementData`中首个为`null`的元素，返回`true`。若传入的要移除的元素不为`null`，遍历`elementData`，找到要移除的元素并移除。若要移除的元素并不存在于`elementData`，那么返回`false`。

```java
/**
* Appends all of the elements in the specified collection to the end of
* this list, in the order that they are returned by the
* specified collection's Iterator.  The behavior of this operation is
* undefined if the specified collection is modified while the operation
* is in progress.  (This implies that the behavior of this call is
* undefined if the specified collection is this list, and this
* list is nonempty.)
*
*/
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}

/**
* Inserts all of the elements in the specified collection into this
* list, starting at the specified position.  Shifts the element
* currently at that position (if any) and any subsequent elements to
* the right (increases their indices).  The new elements will appear
* in the list in the order that they are returned by the
* specified collection's iterator.
*
*/
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                        numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```
`addAll(Collection<? extends E> c)`是将集合`c`追加到`elementData`尾部，`addAll(int index, Collection<? extends E> c)`是将集合`c`插入到指定位置。若指定的`index`为`elementData`的`size`处，那么直接将`c`追加到`elementData`尾部。否则，需要先将`elementData`中以`index`下标开始的`numMoved`个元素后移一位，然后把`c`插入`index`处。

```java
/**
* Removes from this list all of the elements whose index is between
* {@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
* Shifts any succeeding elements to the left (reduces their index).
* This call shortens the list by {@code (toIndex - fromIndex)} elements.
* (If {@code toIndex==fromIndex}, this operation has no effect.)
*/
protected void removeRange(int fromIndex, int toIndex) {
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
`removeRange`将指定范围内的元素移除，首先将`toIndex`之后的元素拷贝到`fromIndex`之后，调整`elementData`的`size`为`newSize`，然后将`newSize`之后的元素全部置空供`GC`回收。

```java
/**
* Removes from this list all of its elements that are contained in the
* specified collection.
*/
public boolean removeAll(Collection<?> c) {
    return batchRemove(c, false);
}

/**
* Retains only the elements in this list that are contained in the
* specified collection.  In other words, removes from this list all
* of its elements that are not contained in the specified collection.
*/
public boolean retainAll(Collection<?> c) {
    return batchRemove(c, true);
}

private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                            elementData, w,
                            size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```
`removeAll`的作用是删除`elemetData`中与指定集合`c`相同的元素。`retainAll`的作用是保留`elementData`中与指定集合`c`相同的元素，删除其他的元素。`removeAll`和`retainAll`传入一个`boolean`类型的`complement`，在`batchRemove`中通过`if (c.contains(elementData[r]) == complement) `来决定是否保留元素。`removeAll`传递过来的`complement`为`false`，因此当`elementData`中的元素不包含在`c`中时，保留这个元素，达到了`remove`的效果。`retainAll`传递过来的`complement`为`true`，因此当`elementData`中的元素包含在`c`中，保留这个元素，达到了`retain`的效果。

## 总结

注意`ArrayList`每次在增加元素前都需要调用`ensureCapacityInternal`来确保`elementData`的容量足够，当容量不够时，调用`ensureExplicitCapacity`，`ensureExplicitCapacity`调用`grow`方法线将容量增加到原来容量的`1.5`倍，若果增加为原来的`1.5`倍之后容量还不够，那么就把新容量直接设置为传递过来的容量(`minCapacity`)。然后调用`Arrays.copyOf`将原来的元素拷贝到新数组中。可以看出，当容量不够时，增加元素变成了一个非常耗时的过程。因此建议在实现能够确定元素数量的情况下才使用`ArrayList`，否则更加建议使用`LinkedList`。

`ArrayList`的源码中多次调用了两个数组拷贝的方法，分别是`Arrays.copyOf`和`System.arraycopy`，查看这两个方法的源码。

```java
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
```
该`copyOf`方法调用了另一个`copyOf`方法：

```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                    Math.min(original.length, newLength));
    return copy;
}
```

该方法创建了一个新的数组，然后调用`System.arraycopy`将元素拷贝到新数组中。

```java
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

`arraycopy`是一个本地方法，通过调用系统的`C/C++`方法实现。该方法可以保证在同一个数组内的元素的复制和移动。

通过源码的分析，可以更加深刻的理解为什么说`ArrayList`的查找效率高而增加、删除元素的效率低。




