## 并发工具类

###### 模仿CopyOnWriteArrayList实现的 CopyOnWriteMap

```java
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }

    public V put(K key, V value) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }

    public V get(Object key) {
        return internalMap.get(key);
    }

    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```

- 适合读多写少的场景。读写是分离的，遍历和修改操作是作用在不同的容器上，所以在迭代器遍历时不会抛出异常

- 每次写操作会对原容器拷贝一份，数据量大的时候性能不好，内存压力大，可能引起频繁fullGC

- 读的是老容器的数据。如果希望写入的数据立马能准确读取，不要用CopyOnWrite

