# SparsArray

## 1、成员变量

```java
private static final Object DELETED = new Object();
private boolean mGarbage = false;
//存储key
private int[] mKeys;
//存储value
private Object[] mValues;
//存储元素数量
private int mSize;
```



## 2、android.util.SparseArray#put

```java
public void put(int key, E value) {
    //传入key数组、当前元素数量，以及key
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

static int binarySearch(int[] array, int size, int value) {
    //左指针
    int lo = 0;
    //右指针
    int hi = size - 1;

    while (lo <= hi) {
        //向右移动一位 与/2效果相同， 可以学习一下
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

模拟一遍插入过程：

* put(1, "a")

  1. binarySearch(key[], 0, 1) 返回结果~1 = -1   mKeys[1] = 1, mValues[1] = "a";
* put(2, "b")
  1. binarySearch(key[], 1, 2) 

