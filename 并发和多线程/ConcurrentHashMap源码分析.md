# ConcurrentHashMap源码分析

## initialTable方法

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl如果<0说明，初始化正在被别的线程持有
        if ((sc = sizeCtl) < 0)
            // 礼让，等待下次调度再来尝试获取sizeCtl来进行自己的工作
            Thread.yield(); // lost initialization race; just spin
        
        // if sizeCtl 不是负数，那么我可以进行初始化操作，并且我把sizeCtl置为-1，这里使用的是cas 操作 ， SIZECTL为sizeCtl变量的内存偏移量
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 如果table为空
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                // 初始化完成后 重新置为负数
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## get

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 获取key的hashcode
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 通过使用(hashCode & (length - 1))的计算方法，获得该记录在table中的index
        // tableAt方法根据索引找到tab中的node元素 e，如果没有找到直接返回null
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // e是table槽中的第一个结点，如果第一个结点的hashcode正好等于key的hashcode
        if ((eh = e.hash) == h) {
            // 如果e的key正好和传入的key相等，前面的判断可能都为null，这是也相等
            // 所以需要后面的判断，如果不为null也相等，才说明key的值才真正的相等
            if ((ek = e.key) == key） || (ek != null && key.equals(ek)))
                // 这种情况直接返回e的value
                return e.val;
        }
        // eh < 0 说明是红黑树
        else if (eh < 0)
            // 调用红黑树的查找方法：find(h, key)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 走到这一步， 说明e的位置是一条链表，遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```