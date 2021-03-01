---
title: 最大堆和最小堆 golang实现
date: 2019-01-15 22:32:24
tags:
- golang
---

# 最小堆和最大堆 golang实现

二叉堆是一种特殊的堆，它满足两个性质：结构性和堆序性

- 结构性：二叉堆是一颗完全二叉树，完全二叉树可以用一个数组表示，不需要指针，所以效率更高。当用数组表示时，数组中任一位置i上的元素，其左儿子在位置2i上，右儿子在位置(2i+ 1)上，其父节点在位置(i/2)上。
- 堆序性质：堆的最小值或最大值在根节点上，所以可以快速找到最大值或最小值。

最大堆和最小堆是二叉堆的两种形式。
-最大堆：根结点的键值是所有堆结点键值中最大者的堆。
-最小堆：根结点的键值是所有堆结点键值中最小者的堆。

# 1. 最小堆实现，不使用container/heap

``` go
type MinHeap struct {
    Element []int
}
```

## 定义构造方法

数组中第一个元素不使用，存放一个小于堆中任何数字的值用于结束循环。

```go
// MinHeap构造方法
func NewMinHeap() *MinHeap {
    // 第一个元素仅用于结束insert中的 for 循环
    h := &MinHeap{Element: []int{math.MinInt64}}
    return h
}
```

## 插入

插入元素就直接将元素增加到堆的末尾，然后进行上浮操作，使二叉堆有序。
如果上浮一直到根，时间复杂度为O(log N)，但这种上浮操作一般结束的要早。

```go
// 插入数字,插入数字需要保证堆的性质
func (H *MinHeap) Insert(v int) {
    H.Element = append(H.Element, v)
    i := len(H.Element) - 1
    // 上浮
    for ; H.Element[i/2] > v; i /= 2 {
        H.Element[i] = H.Element[i/2]
    }

    H.Element[i] = v
}
```

## 删除最小值

删除最大元素就直接从二叉堆顶端删除，然后进行下沉操作。最坏时间复杂度同样为O(log N)


```go
// 删除并返回最小值
func (H *MinHeap) DeleteMin() (int, error) {
    if len(H.Element) <= 1 {
        return 0, fmt.Errorf("MinHeap is empty")
	}
	minElement := H.Element[1]
	lastElement := H.Element[len(H.Element)-1]
	var i, child int
	for i = 1; i*2 < len(H.Element); i = child {
		child = i * 2
		if child < len(H.Element)-1 && H.Element[child+1] < H.Element[child] {
			child ++
		}
		// 下滤一层
		if lastElement > H.Element[child] {
			H.Element[i] = H.Element[child]
		} else {
			break
		}
	}
	H.Element[i] = lastElement
	H.Element = H.Element[:len(H.Element)-1]
	return minElement, nil
}
```

## 其他方法

```go
// 堆的大小
func (H *MinHeap) Length() int {
	return len(H.Element) - 1
}

// 获取最小堆的最小值
func (H *MinHeap) Min() (int, error) {
	if len(H.Element) > 1 {
		return H.Element[1], nil
	}
	return 0, fmt.Errorf("heap is empty")
}

// MinHeap格式化输出
func (H *MinHeap) String() string {
	return fmt.Sprintf("%v", H.Element[1:])
}

```

# 2.下面介绍container/heap包 和最大堆的实现

## heap源码中定义了一个Interface 的接口，此接口一共包含五个方法，我们定义一个实现此接口的类就实现了一个二叉堆

container/heap/heap.go
```go
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
```

sort.go
```go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

## 定义一个最大堆，并实现heap.Interface 接口
```go
type MaxHeap []int

func (h MaxHeap) Len() int {
	return len(h)
}

func (h MaxHeap) Less(i, j int) bool {
	// 由于是最大堆，所以使用大于号
	return h[i] > h[j]
}

func (h *MaxHeap) Swap(i, j int) {
	(*h)[i], (*h)[j] = (*h)[j], (*h)[i]
}

func (h *MaxHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

// Pop 弹出最后一个元素
func (h *MaxHeap) Pop() interface{}{
	res := (*h)[len(*h)-1]
	*h = (*h)[:len(*h)-1]
	return res
}

```
## 测试最大堆
```go
func main() {
	h := make(MaxHeap, 0)
	heap.Init(&h)

	heap.Push(&h, 8)
	heap.Push(&h, 1)
	heap.Push(&h, 4)
	heap.Push(&h, 5)
	heap.Push(&h, 2)

	fmt.Println(heap.Pop(&h))
	fmt.Println(heap.Pop(&h))
	fmt.Println(heap.Pop(&h))
	fmt.Println(heap.Pop(&h))
	fmt.Println(heap.Pop(&h))

}

>>>
8
5
4
2
1
```
