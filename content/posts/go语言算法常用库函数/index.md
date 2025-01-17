---
title: "Go 语言算法常用库函数"
date: "2024-03-10"
---

时至今日已经用 Go 刷了大半年的算法了，对于大部分题型都做过了，因为我一直保持着网页没有语法提示的情况下进行刷算法，所以有些必须记在脑子里才能在关键时刻用出来，趁此机会总结回顾一下 Go 语言算法常用库函数。

## 常量

最大值和最小值（其他不同于 int 的常量同理）

```
math.MaxInt , math.MinInt
```

## 排序

（这部分慎用，很多直接考排序的算法在面试的时候是肯定不允许使用的，在一些排序不是重点的题里面可以用，比如合并数组）

```
sort.Sort(data Interface)

//需要实现以下方法：
Len() int 
Less(i int, j int) bool 
Swap(i int, j int)
```

```
sort.Ints(a []int)

//Ints函数将a排序为递增顺序。
```

```
sort.Strings(a []string)

//Strings函数将a排序为递增顺序。
```

```
func SearchInts(a []int, x int) int

//SearchInts在递增顺序的a中搜索x，返回x的索引。如果查找不到，返回值是x应该插入a的位置（以保证a的递增顺序），返回值可以是len(a)。
```

```
func SearchStrings(a []int, x int) int

//SearchStrings在递增顺序的a中搜索x，返回x的索引。如果查找不到，返回值是x应该插入a的位置（以保证a的递增顺序），返回值可以是len(a)。
```

```
// 个人用的最多
func Slice(x any, less func(i int, j int) bool)

people := []struct {     
    Name string     
    Age  int 
}{ {"Gopher", 7}, {"Alice", 55}, {"Vera", 24},{"Bob", 75}, } 


sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name }
```

## 数据结构

### 链表

直接考链表的算法题用不着官方库，也不能用，但是例如 LRU 算法中使用官方实现的链表就会减轻我们的负担

创建链表

```
l := list.New()
```

list包实现了双向链表。要遍历一个链表：

```
for e := l.Front(); e != nil; e = e.Next() {
        // do something with e.Value
}
```

获取链表长度

```
func (l *List) Len() int
```

获取链表第一个元素

```
func (l *List) Front() *Element
```

获取链表最后一个元素

```
func (l *List) Back() *Element
```

PushFront将一个值为v的新元素插入链表的第一个位置，返回生成的新元素。

```
func (l *List) PushFront(v interface{}) *Element
```

PushBack将一个值为v的新元素插入链表的最后一个位置，返回生成的新元素。

```
func (l *List) PushBack(v interface{}) *Element
```

MoveToFront将元素e移动到链表的第一个位置，如果e不是l的元素，l不会被修改。

```
func (l *List) MoveToFront(e *Element)
```

MoveToBack将元素e移动到链表的最后一个位置，如果e不是l的元素，l不会被修改。

```
func (l *List) MoveToBack(e *Element)
```

Remove删除链表中的元素e，并返回e.Value。

```
func (l *List) Remove(e *Element) interface{}
```

### 堆

在获取xx中第k大/小的算法题用的很多，当然如果你能手撕堆排序那就很强了

**堆的初始化：**

heap包提供了对任意类型（实现了heap.Interface接口）的堆操作。（最小）堆是具有“每个节点都是以其为根的子树中最小值”属性的树。

树的最小元素为其根元素，索引0的位置。

heap是常用的实现优先队列的方法。要创建一个优先队列，实现一个具有使用（负的）优先级作为比较的依据的Less方法的Heap接口，如此一来可用Push添加项目而用Pop取出队列最高优先级的项目。

```
type Interface interface{
    sort.Interface
    Push(x interface{}) // 向末尾添加元素
    Pop() interface{}   // 从末尾删除元素   
}

//sort.Interface
type Interface interface{
    // Len方法返回集合中的元素个数      
    Len() int
    // Less方法报告索引i的元素是否比索引j的元素小
    Less(i, j int) bool
    // Swap方法交换索引i和j的两个元素
    Swap(i, j int)    
}
```

**需要实现以上5个方法**

向堆h中插入元素x，并保持堆的约束性。复杂度O(log(n))，其中n等于h.Len()。

```
func Push(h Interface, x interface{})
```

删除并返回堆h中的最小元素（不影响约束性）。复杂度O(log(n))，其中n等于h.Len()。等价于Remove(h, 0)。

```
func Pop(h Interface) interface{}
```

删除堆中的第i个元素，并保持堆的约束性。复杂度O(log(n))，其中n等于h.Len()。

```
func Remove(h Interface, i int) interface{}
```

在修改第i个元素后，调用本函数修复堆，比删除第i个元素后插入新元素更有效率。

```
func Fix(h Interface, i int)
```

### 环

基本没用过，就不说了

## 字符串

### strings 包

常见的比如 split、join 就不说了

判断字符串s是否包含子串substr。(考KMP但是你不幸地忘记的时候可以用这个函数）

```
func Contains(s, substr string) bool
```

子串sep在字符串s中第一次出现的位置，不存在则返回-1。

```
func Index(s, sep string) int
```

s中第一个满足函数f的位置i（该处的utf-8码值r满足f(r)==true），不存在则返回-1。

```
func IndexFunc(s string, f func(rune) bool) int
```

### strconv 包

用的多的就两个吧，其他的强转一般也能搞定

```
func ParseInt(s string, base int, bitSize int) (i int64, err error)
```

```
func Atoi(s string) (i int, err error)
```
