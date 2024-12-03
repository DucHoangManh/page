---
title: "Go: Giới thiệu về type parameter (generic) qua ví dụ"
date: 2022-06-30T20:21:52+07:00
tags: ["go", "efective-go", "generic"]
---

### 1. Kiểm tra sự xuất hiện của một phần tử trong slice
Type parameter hay generic là một tính năng mới xuất hiện từ phiên bản 1.18 của Go. Với generic, ta có thể viết các hàm hoặc type có thể dùng được cho nhiều kiểu dữ liệu đầu vào khác nhau mà không cần phải lặp lại code nhiều lần.   
Hãy bắt đầu với một ví dụ đơn giản và phổ biến: Kiểm tra xem string có xuất hiện trong slice hay không  
```go
func StringSliceContains(ss []string, match string) bool {
	for _, s := range ss {
		if s == match {
			return true
		}
	}
	return false
}
```
Rõ ràng là trong quá trình sử dụng thực tế, ta sẽ gặp các usecase tương tự với các kiểu dữ liệu khác. Trước khi có generic, việc phổ biến nhất sẽ là viết riêng các hàm cụ thể cho từng kiểu dữ liệu `IntSliceContains()`, `Int64SliceContains()`... bên cạnh việc sử dụng code generation hay reflection.   
Có thể thấy các hàm này đều lặp lại một logic giống nhau, chỉ khác ở kiểu dữ liệu đầu vào, với generic, ta hoàn toàn có thể rút gọn lại các hàm này lại thành một, rất gọn và dễ dàng bảo trì về sau:   
```go
func SliceContains[T comparable](ss []T, match T) bool {
	for _, s := range ss {
		if s == match {
			return true
		}
	}
	return false
}
```
Giải thích:  
`[T comparable]` được gọi là type parameter và có thể sử dụng được với một `func` hoặc một `type`, trong trường hợp này ám chỉ việc hàm `SliceContains` có thể nhận vào đầu vào kiểu T thỏa mãn ràng buộc `comparable`.   
Ràng buộc ở đây là một interface mà T cần thỏa mãn, interface này có thể chứa method signature thường thấy hoặc tập hợp các type bất kì, xem thêm [package constraints](https://pkg.go.dev/golang.org/x/exp/constraints).   
Ta hoàn toàn có thể  định nghĩa constraints cùa riêng mình.
```go
type MyConstraint interface {
    ~int | ~string
}

func SliceContains[T MyConstraint](ss []T, match T) bool {
	for _, s := range ss {
		if s == match {
			return true
		}
	}
	return false
}
```
Như ví dụ trên đây, T cần là `int` hoặc `string`, toán tử `~` tức ràng buộc thỏa mãn với cả các kiểu được định nghĩa từ `int` hoặc `string` (ví dụ `type Name string`, `type Age int`)
```go
func TestSliceContains(t *testing.T) {
	intSlice := []int{4, 5, 6}
	stringSlice := []string{"a", "abb", "c"}
	fmt.Println(SliceContains(intSlice, 5)) // true
	fmt.Println(SliceContains(stringSlice, "c")) // true
}
```
Có thể thấy là khi gọi hàm không cần chỉ rõ ra T là kiểu gì mà chỉ cần truyền các parameter vào, go có thể tự hiểu và thực thi đúng hàm mà ta mong muốn, `SliceContains[int](intSlice, 5)` cũng tương đương với `SliceContains(intSlice, 5)`.  
Ta cũng có thể viết một hàm kiểm tra phần tử có thuộc slice hay không mà không có ràng buộc, thay vào đó, truyền vào một funcion `equalFunc` để sử dụng thay cho toán tử `==`:
```go
func SliceContainsWithEqual[T any](ss []T, match T, equalFunc func(T, T) bool) bool {
	for _, s := range ss {
		if equalFunc(s, match) {
			return true
		}
	}
	return false
}
```
### 2. Generic wrapper cho container/heap
Trong go, khi cần sử dụng heap, ta có thể sử dụng package `container/heap` và implement `heap.Interface` cho kiểu dữ liệu muốn sử dụng.   
Ví dụ một min heap cho kiểu int từ trong document của `container/heap`:
```go
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}
```

Trong nhiều trường hợp, ta chỉ quan tâm tới kiểu dữ liệu và method `Less`, còn cách triền khai tương tự như ví dụ trên.  
Có thể dùng type parameter để đơn giản hóa việc sử dụng heap như sau:
```go
type Heap[T any] struct {
	data     []T
	lessFunc func(a, b T) bool
}

func (b *Heap[T]) Len() int {
	return len(b.data)
}

func (b *Heap[T]) Less(i, j int) bool {
	return b.lessFunc(b.data[i], b.data[j])
}

func (b *Heap[T]) Swap(i, j int) {
	tmp := b.data[i]
	b.data[i] = b.data[j]
	b.data[j] = tmp
}

func (b *Heap[T]) Push(x any) {
	b.data = append(b.data, x.(T))
}

func (b *Heap[T]) Pop() any {
	old := b.data
	oldLen := len(old)
	res := old[oldLen-1]
	b.data = b.data[:oldLen-1]
	return res
}

func New[T any](lessFunc func(T, T) bool) *Heap[T] {
	h := &Heap[T]{
		lessFunc: lessFunc,
	}
	heap.Init(h)
	return h
}
```
Type parameter cũng có thể sử dụng với `type` như trong ví dụ này.
Mấu chốt trong cách triển khai này chính là lưu lại `lessFunc` trong struct `Heap`, do ta không thể thay đổi method `Less` tại runtime, thay vào đó `Less` sẽ gọi vào hàm được lưu trong receiver.  
Tuy nhiên có thể thấy rằng kiểu của các method như `Push` hay `Pop` lại là `any` và để đảm bảo implement `heap.Interface`, ta không thể sửa signature của các method này. Ngoài ra, việc gọi trực tiếp `*Heap.Pop` thay vì `heap.Pop(*Heap)`, và `*Heap.Push(T)` thay vì `heap.Push(*Heap, T)` sẽ gây ra kết quả sai, thay vào đó nên tránh cho phép gọi trực tiếp các method này.     
Ta sẽ wrap type mà implement heap interface trong một public Type mà sẽ nhận và trả về kết quả mong muốn:  
Chuyển triển khai trên thành một unexported struct với tên `base`:
```go
type base[T any] struct {
	data     []T
	lessFunc func(a, b T) bool
}

func (b *base[T]) Len() int {
	return len(b.data)
}

func (b *base[T]) Less(i, j int) bool {
	return b.lessFunc(b.data[i], b.data[j])
}

func (b *base[T]) Swap(i, j int) {
	tmp := b.data[i]
	b.data[i] = b.data[j]
	b.data[j] = tmp
}

func (b *base[T]) Push(x any) {
	b.data = append(b.data, x.(T))
}

func (b *base[T]) Pop() any {
	old := b.data
	oldLen := len(old)
	res := old[oldLen-1]
	b.data = b.data[:oldLen-1]
	return res
}
```

Và viết thêm một genetic type với các method mà ta mong muốn:
```go
type Heap[T any] struct {
	base *base[T]
}

func New[T any](lessFunc func(T, T) bool) *Heap[T] {
	b := &base[T]{
		lessFunc: lessFunc,
	}
	heap.Init(b)
	return &Heap[T]{base: b}
}

func (h *Heap[T]) Push(t T) {
	heap.Push(h.base, t)
}

func (h *Heap[T]) Pop() T {
	if h.Len() == 0 { // thêm validate cho Pop
		var t T
		return t
	}
	return heap.Pop(h.base).(T) // chuyển đổi kiểu trả về cho đúng
}

func (h *Heap[T]) Len() int {
	return h.base.Len()
}
```
Chạy thử:
```go
func TestGenericHeap(t *testing.T) {
	data := []int{0, 5, 4, 2, -5, 8, 9, -10}
	lessFunc := func(i, j int) bool {
		return j-i > 0
	}
	heap := New(lessFunc)
	for _, v := range data {
		heap.Push(v)
	}
	for heap.Len() > 0 {
		fmt.Print(heap.Pop(), " ")
	}
}
```
Kết quả:
```
-10 -5 0 2 4 5 8 9
```
Clear hơn khá nhiều phải không? Có thể xem code đầy đủ tại [đây](https://github.com/duchng/generic-heap)

### 3. Khi nào dùng type parameter?
- Khi dùng type parameter chỉ để gọi method của constraints thì không nên dùng type parameter mà nên dùng hàm bình thường nhận vào interface.
thay vì 
```go
func foo[T io.Writer](w T) {
    ...
    w.Write...
    ...
}
```
tốt hơn là 
```go
func foo(w io.Writer) {
    ...
    w.Write...
    ...
}
```
như cách mà nó vẫn được viết từ trước khi go có generic.
- Và trong nhiều trường hợp, việc sử dụng type parameter có thể ảnh hưởng nhiều đến tính dễ bảo trì của code, hãy cân nhắc sử dụng các phương pháp khác đã nêu ở phần 1. Hãy nhớ là go developer vẫn sống (khá) tốt trong một thời gian dài mà không có generic (◍•ᴗ•◍)👍.

