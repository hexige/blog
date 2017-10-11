---
layout:     post
title:      "SparseArray"
date:       2017-10-11 15:45:00
author:     "hxg_"
comments:	true
header-img: "img/post-bg-01.jpg"
---

## SparseArray

内存效率高于HashMap:

- 避免了对key的自动装箱；
- 数据结构不依赖于额外的Entry对象；

避免对key的自动装箱，是因为实际存储key的结构为一个int数组:

```
 private int[] mKeys;
 private Object[] mValues;
```

因其对key的查找为`二分查找`，所以不建议在大量数据的时候使用。

```
// This is Arrays.binarySearch(), but doesn't do any argument validation.
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
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

当remove key的时候，并不会马上压缩数组，会将removed entry标记为删除。此entry可被相同的key重用，或者在收到gc信号的时候删除。

SparseArray同HashMap一样，可在构造方法中定义capacity，默认构造方法为10。

```
public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

所有对元素的操作，均先使用二分查找，判断当前key是否存在。

`get`方法，当前key查找失败或为删除状态，则返回valueIfKeyNotFound，默认返回null。

```
public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i];
        }
    }
```

`delete`方法，若当前key查找成功，并且状态不为删除，则置为删除状态。删除状态的标志即将当前key对应的value赋值为`DELETED`，同时将`mGarbage`置为true;
`mGarbage`变量在`remove`,`delete`等操作时，都会置为true，表示有垃圾产生。在`gc()`和`clear()`后，会将其置为false。

```
public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    
```

其中的`DELETED`为局部变量

```
private static final Object DELETED = new Object();
```

`put`方法：若key查找成功，则直接对相应的value赋值；否则，对查找到的i进行位运算:

```
i = ~i;
```

如果`i < size`并且对应的value为删除状态，则直接赋值（对应了前面说的重复使用）

如果有垃圾`mGarbage`并且当前长度大于key数组的长度，则调用`gc()`移除标记位删除状态的entry。gc完成后再插入新的key和value； 

```
public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            mValues[i] = value;
        } else {
            i = ~i;

            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```


