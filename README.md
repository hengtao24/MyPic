# 1. 前言
Android开发过程中，使用集合的频率很高，但由于Android平台内存的限制，很多时候使用原生Java集合API没有很好的性能，容易消耗内存，所以Android平台也有其集合API，**如SparseArray**、**ArrayMap**。
在某些情况，这两种集合用来代替Java中的**HashMap**能带来更好的性能提升。因此，**本文将结合源码先介绍SparseArray**。

# 2. 简介
**SparseArray<E>** 是用来替换HashMap的数据结构，更准确的是用于替代key为int类型，value为Object类型的HashMap。
由于**SparseArray<E>** 中key被指定为int类型，因此可以节省空间和**int-Integer**的装箱拆箱操作带来的性能消耗。
其内部实现是基于两个数组，mKeys用来保存每个item的key值，mValues为Object类型数组，用来保存value。
```java
private int[] mKeys;
private Object[] mValues;
```
# 3. 构造函数
```java
// 用于标记value数组被删除的元素
private static final Object DELETED = new Object();
// 是否需要GC 
private boolean mGarbage = false;

// 存储key的数组
private int[] mKeys;
// 存储value的数组
private Object[] mValues;
// 集合大小
private int mSize;

// 默认构造函数，初始化容量为10
public SparseArray() {
    this(10);
}

// 指定初始容量
public SparseArray(int initialCapacity) {
    // 初始容量为0的话，就赋值两个轻量级的引用
    if (initialCapacity == 0) {
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        // 初始化对应长度的数组
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length];
    }
    // 集合大小为0
    mSize = 0;
}
```
**SparseArray**为了提升性能，在删除操作时做了一些优化：
当删除一个元素时，并不是立即从value数组中删除它，并压缩数组，
而是将其在value数组中标记为已删除(上述代码中的**DELETED**)。这样当存储相同的key的value时，可以重用这个空间。
如果此空间没有被重用，随后会在合适的时机执行GC操作，将数组压缩避免空间浪费。

# 4. 增/删/查/改
在添加、删除、查找数据的时候都是先使用二分查找法得到相应的index，然后通过index来进行添加、查找、删除等操作。
因为其会对key从小到大排序，使用二分法查询key对应在数组中的下标，因此它不适合大容量的数据存储,比传统的HashMap时间效率低。
## 4.1 增/改
```java
public void put(int key, E value) {
    // 利用二分查找，找到待插入key的下标index
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    // 返回的index是正数，说明之前这个key存在，直接覆盖value即可
    if (i >= 0) {
        // 修改该值
        mValues[i] = value;
    } else {
        // 若返回的index是负数，说明key不存在
        // 先对返回的i取反，得到应该插入的位置i
         i = ~i;
        // 如果i没有越界，且对应位置是已删除的标记，则复用这个空间
        if (i < mSize && mValues[i] == DELETED) {
            // 赋值
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        // 如果需要GC且需要扩容
        if (mGarbage && mSize >= mKeys.length) {
            // 先触发GC
            gc();

            // gc后，下标i可能发生变化，所以再次用二分查找找到应该插入的位置i
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }
        // 插入key(可能需要扩容)
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        // 插入value(可能需要扩容)
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        // 增加集合大小
        mSize++;
    }
}

// 二分查找
static int binarySearch(int[] array, int size, int value) {
    int lo = 0;
    int hi = size - 1;

    while (lo <= hi) {
        final int mid = (lo + hi) >>> 1;
        final int midVal = array[mid];

        if (midVal < value) {
            lo = mid + 1;
        } else if (midVal > value) {
            hi = mid - 1;
        } else {
            return mid;
        }
    }
    // 若没找到，则lo是value应该插入的位置，是一个正数。对这个正数取反，返回负数回去
    return ~lo;
}

// gc操作，压缩数组
private void gc() {
    // 保存压缩前数组大小
    int n = mSize;
    // 新数组的下标
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;
    // 遍历集合压缩
    for (int i = 0; i < n; i++) {
        Object val = values[i];
        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                // 置为null，防止内存泄漏
                values[i] = null;
            }
            o++;
        }
    }
    // 修改标记
    mGarbage = false;
    // 修改新集合的大小
    mSize = o;
}

// GrowingArrayUtils.insert辅助类
public static int[] insert(int[] array, int currentSize, int index, int element) {
    // 断言 确认当前集合长度小于等于array数组长度
    assert currentSize <= array.length;
    // 无需扩容
    if (currentSize + 1 <= array.length) {
        // 将array数组内元素，从index开始 后移一位
        System.arraycopy(array, index, array, index + 1, currentSize - index);
        array[index] = element;
        return array;
    }
    // 需要扩容
    // 构建新的数组
    // growSize()确定新的size
    int[] newArray = new int[growSize(currentSize)];
    // 将原数组中index之前的数据复制到新数组中
    System.arraycopy(array, 0, newArray, 0, index);
    // 在index处赋值
    newArray[index] = element;
    // 将原数组中index及其之后的数据赋值到新数组中
    System.arraycopy(array, index, newArray, index + 1, array.length - index);
    return newArray;
}

// 根据现在的size，返回合适的扩容后的容量
public static int growSize(int currentSize) {
    // 如果当前size小于等于4，则返回8，否则返回当前size的两倍
    return currentSize <= 4 ? 8 : currentSize * 2;
}
```

## 4.2 删
### 4.2.1 根据key值删除
```java
// 根据指定key值删除
public void remove(int key) {
    delete(key);
}

// 删除操作
public void delete(int key) {
   // 二分查找待删除key值的index
   int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

   // 如果该下标存在，则先标记为DELETED
   if (i >= 0) {
       if (mValues[i] != DELETED) {
           mValues[i] = DELETED;
           // 修改是否需要gc标记
           mGarbage = true;
       }
   }
}
```
### 4.2.2 根据index值删除
```java
// 根据index值删除
public void removeAt(int index) {
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}
```
### 4.2.3 根据index范围值删除
```java
// 删除某一index范围内的值
public void removeAtRange(int index, int size) {
    final int end = Math.min(mSize, index + size);
    for (int i = index; i < end; i++) {
        removeAt(i);
    }
}
```

## 4.3 查
### 4.3.1 根据key值查询
```java
// 查询，key不存在则返回null
public E get(int key) {
    return get(key, null);
}
// 查询，带默认参数
public E get(int key, E valueIfKeyNotFound) {
    // 查找下标
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
```
### 4.3.2 根据index值查询
```java
// 查询key值
public int keyAt(int index) {
    // 是否需要GC
    if (mGarbage) {
        gc();
    }
    return mKeys[index];
}

// 查询value值
public E valueAt(int index) {
    // 是否需要GC
    if (mGarbage) {
        gc();
    }
    return (E) mValues[index];
}
```

### 4.3.3 查询下标
```java
// 查询某一key值下标
public int indexOfKey(int key) {
    // 是否需要GC
    if (mGarbage) {
        gc();
    }
    // 二分查找下标
    return ContainerHelpers.binarySearch(mKeys, mSize, key);
}

// 查询某一value值下标
public int indexOfValue(E value) {
    // 是否需要GC
    if (mGarbage) {
        gc();
    }
    // 遍历查询，一个value对应多个key值优先返回第一个
    for (int i = 0; i < mSize; i++)
        if (mValues[i] == value) {
            return i;
        }
    }
    // 找不到返回-1
    return -1;
}
```

# 5. 总结
- **SparseArray**适合使用数据量不大（千以内），空间比时间重要，需要使用Map，且key为int类型的场景
- 扩容时，当前容量小于等于4，则扩容后容量为8.否则为当前容量的两倍。扩容因子和**ArrayList，ArrayMap**不同(扩容一半)，和**Vector**相同(扩容一倍)
- 扩容时的数组操作也是通过复制数组实现的，类似于**ArrayList**扩容的操作
- 除了**SparseArray，Android**还有其它的一些类似的数据结构，是用于存放基本数据类型的键值对：
	SparseIntArray — int:int
	SparseBooleanArray— int:boolean
	SparseLongArray— int:long
