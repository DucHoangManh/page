---
title: "Design pattern thân thiện trong go"
date: 2021-12-26T22:14:29+07:00
tags: ["go", "design pattern", "efective-go"]
---

### Tại sao lại có bài viết này?:

Design pattern: Những giải pháp có thể tái sử dụng cho các vấn đề thường gặp tại một ngữ cảnh nhất định trong quá trinh thiết kế phần mềm.  

Bài viết này nói về một số  Design pattern "thân thiện" hơn trong go, được mình tổng hợp dựa trên buổi talk của Ryan Djurovich (https://www.youtube.com/watch?v=HHqv3_rUr88) và một số nguồn tài liệu khác mà mình đọc được.

## Factory pattern
Khỏi phải nói về độ phổ biến của nó rồi, rất hữu ích khi cần phải khởi tạo một đối tượng có nhiều triển khai (concrete types), phía client chỉ cần quan tâm đến các method, Factory sẽ lo việc lựa chọn triển khai nào sẽ được dùng để xử lý dữ liệu.

```go
// use Stringer as the interface

type ErrPrint struct{}

func (p *ErrPrint) String() string {
	return "some error happens"
}

type InfoPrint struct{}

func (p *InfoPrint) String() string {
	return "nothing dramatically happens"
}

func NewPrinter(kind string) (result fmt.Stringer, err error) {
	switch kind {
	case "error":
		result = &ErrPrint{}
	case "info":
		result = &InfoPrint{}
	default:
		err = errors.New("invalid kind")
	}
	return
}
```

## Decorator (Functional option)
Để nói về pattern này, đầu tiên hãy xem ví dụ bên dưới: 
```go
func runServer() {
	http.HandleFunc("/", helloEndpoint)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
func helloEndpoint(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello world")
}
```
Đoạn code trên tạo ra một web server đơn giản với net/http package với một endpoint duy nhất.
Bài toán được đặt ra là bây giờ cần log lại thời gian cần để hoàn tất xử lý request. Để gia tăng tính tái sử dụng, tránh phải sửa lại handler, cũng như dễ dàng bảo trì về sau, ta có thể viết như sau:
```go
func runServer() {
	http.HandleFunc("/", durationLogger(helloEndpoint))
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func helloEndpoint(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello world")
}

func durationLogger(f http.HandlerFunc) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		startTime := time.Now()
		f(writer, request)
		log.Printf("complete handle request after %s ms", time.Since(startTime).Milliseconds())
	}
}
```

Như vậy, chúng ta có thể dễ dàng thêm tính năng log thời gian này vào những handler cần thiết, việc thay đổi nội dung log cũng trở nên dễ dàng hơn.
Đây chỉ là một ví dụ đơn giản về middleware, trong thực tế việc triển khai có thể phức tạp hơn để phù hợp với các nhù cầu khác nhau.

## Iterator 
Tại sao lại cần pattern này thay vì duyệt (array, slice, map, channel...) trong go? 
- Có thể kết hợp với decorator
- Khi viết một module nào đó mà muốn che giấu triển khai bên dưới, chỉ cho phép phía sử dụng duyệt tuần tự các phần tử.
- Ví dụ io.Reader, sql/database.Row

```go
type Iterator struct {
	tasks    []string
	position int
}
// Next will return the next task in the slice
// if there's more data to iterate, more will be true
func (t *Iterator) Next() (pos int, val string, more bool) {
	t.position++
	if t.position > len(t.tasks) {
		return t.position, "", false
	}
	return t.position, t.tasks[t.position-1], true
}
```
Trên đây là một ví dụ đơn giản về triển khai iterator trong go
```go
for _, val, more := i.Next(); more; _, val, more = i.Next() {
	fmt.Println(val)
}
```

## Dependency Injection 
DI trong go có thể được triển khai theo nhiều cách, có cả những thư viện chuyên dùng để DI trong go (google/wire, uber-go/fx), trong bài viết này sẽ chỉ nói vể cách triển khai đơn giản nhất 
```
┌────────────────────┬──────────┬────────────────────────────────┬────────┐
│ Client package     │  Client  ├─────►      <<interface>>       │        │
│                    └──────────┘     │ Client Service Interface │        │
│                                     └─────▲────────────────────┘        │
│                                           │                             │
├───────────────────────────────────────────┼─────────────────────────────┤
├───────────────────────────┼───────────────|─────┼─┼──────────────────┼──┤
│ Service package           │  Concrete Service 1 │ │Concrete Service 2│  │
│                           └─────────────────────┘ └──────────────────┘  │
│                                                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
Đây là một đoạn code ví dụ co dependency injection sử dụng constructor
```go
func main() {
    s := NewMyService(os.Stderr)
    s.WriteHello("world")
}
type MyService struct {
    writer io.Writer
}
func NewMyService(writer io.Writer) MyService {
    return MyService{
        writer: writer,
    }
}
func (s *MyService) WriteHello(m string) {
    fmt.Fprintf(s.writer, "Hello %s\n", m)
}
```
Thêm method để có thể sử dụng setter injection:
```go
func (s *MyService) SetWriter(w io.Writer) {
    s.writer = w
}
```
So sánh hai cách:
- constructor: Đảm bảo dependency được sử dụng luôn valid
- setter: Có thể thay thế concrete service trong quá trình chạy chương trình(runtime)

## Repository
Một pattern trong thiết kế phần mếm hướng tới việc tiến hóa lâu dài của ứng dụng, các module phụ thuộc có thể được thay đổi trong tương lai.
Ví dụ dưới đề thể hiện việc triển khai một ứng dụng mà lớp storage có thể được thay đổi:
`package post` - khai báo model 
```go
type Post struct {
	ID   int64
	Title string
}
```
`package domain` - khai báo các interface client sử dụng để thao tác với storage
```go
// Repository must be implemented by all implementations of Post storage
type Repository interface {
	FindAll() ([]post.Post, error)
	Store(post post.Post) (post.Post, error)
	DeleteById(postId int64) error
}
```
`package memstorage` - một triển khai của storage
```go
//in-memory implementation
type PostStorage struct {
	posts     map[int64]string
	highestID int64
}

func (p *PostStorage) FindAll() ([]post.Post, error) {
	result := make([]post.Post, 0)
	for id, title := range p.posts {
		result = append(result, post.Post{
			ID:    id,
			Title: title,
		})
	}
	return result, nil
}

func (p *PostStorage) Store(post post.Post) (post.Post, error) {
	if p.posts == nil {
		p.posts = make(map[int64]string)
	}
	if post.ID <= 0 {
		p.highestID++
		post.ID = p.highestID
	} else {
		if _, exists := p.posts[post.ID]; !exists {
			return post, fmt.Errorf("post already exist")
		}
	}
	p.posts[post.ID] = post.Title
	return post, nil

}

func (p *PostStorage) DeleteById(postId int64) error {
	delete(p.posts, postId)
	return nil
}
```

Sử dụng repository package:
`package main`
```go
func main() {
	postRepo := memstorage.PostStorage{}
	//may change to other implementation in the future
	newPost, err := postRepo.Store(post.Post{
		Title: "Eagles fly",
	})
	if err != nil {
		log.Println("can not create post", err)
	} else {
		log.Printf("created post with id %d", newPost.ID)
	}
	posts, err := postRepo.FindAll()
	if err != nil {
		log.Println("can not fetch posts", err)
	} else {
		log.Println(posts)
	}
}
```
Bằng cách chia nhỏ việc triển khai, domain interface và model, trong trường hợp một phần mềm cần thay thế các module trong tương lai, có thể update một cách dễ dàng.

Mã nguồn trong bài viết có thể xem tại: https://github.com/duchng/go-patterns




 
