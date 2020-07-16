---
title: "golang实现一个比较简单一致性hash"
date: 2020-07-16T00:44:54+08:00
---

一致性hash通常用来做数据分片，保证平衡性、单调性。直接上代码:

```go
type Hash func(data []byte) uint32
type Map struct {
  hash     Hash   // hash函数
  replicas int   // 这个是虚拟节点的数量
  keys     []int  // hash环上的节点定位
  hashMap  map[int]string // hash环到真实节点的映射
}
```

初始化
```go
func New(replicas int, fn Hash) *Map {
    m := &Map{
        replicas: replicas,
        hash:     fn,
        hashMap:  make(map[int]string),
    }
    if m.hash == nil {
        m.hash = crc32.ChecksumIEEE 
    }
    return m
}
```

添加节点
```go
func (m *Map) Add(keys ...string) {
    for _, key := range keys {
        for i := 0; i < m.replicas; i++ {
            // 这个是生成虚拟节点的方法，曾经一个七牛的人问过我虚拟节点怎么生成，我没回答出来，惭愧
            hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
            m.keys = append(m.keys, hash)
            m.hashMap[hash] = key
        }
    }
    sort.Ints(m.keys)
}
```

节点请求
```go
func (m *Map) Get(key string) string {
    hash := int(m.hash([]byte(key)))
    // 过程很简单，就是从hash环上顺时针方向找到一个节点
    idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })
    if idx == len(m.keys) {
        idx = 0
    }
    return m.hashMap[m.keys[idx]]
}
```