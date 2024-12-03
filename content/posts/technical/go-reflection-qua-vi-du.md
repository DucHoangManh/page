---
title: "Go: Giới thiệu về reflection qua ví dụ"
date: 2022-07-13T00:01:23+07:00
tags: ["go", "efective-go"]
---

## Reflection trong Go  

Reflection trong go giúp cho ta có thể theo dõi code tại thời điểm `runtime`, cho phép tiếp cận mã nguồn chương trình dưới dạng data có thể xử lý thay vì các lệnh có thể thực thi (một nhánh trong `metaprogramming`).  
Reflection trong go có thể được thực hiện thông qua `reflect` package.
Một số khả năng của reflect:
- Kiểm tra thông tin của một struct (số lượng method, số lượng field, đọc struct tag...) mà không cần biết trước về struct đó.
- Kiểm tra và cập nhật một type (slice, channel, struct, interface, pointer...) mà không cần biết trước về type đó.   

Đổi lại những khả năng mạnh mẽ của recflect, performance và tính maintainable của code sẽ giảm, cần cân nhắc kĩ trước khi quyết định đưa vào ứng dụng.
> Clear is better than clever. Reflection is never clear - Go proverb (Rob Pike) 

Ứng dụng của reflect:
- Các function, method với đầu vào không rõ trước.
- Viết công cụ phân tích mã nguồn.
- Thực thi code linh hoạt (VD liệt kê các method của một struct và gọi method theo tên).

Một số package, library sử dụng reflection: `fmt`, `encoding/json`, `gorm`, `sqlx`...

## Các khái niệm quan trọng
Có hai type quan trọng trong reflect package: `reflect.Type` và `reflect.Value`, mọi biến trong một chương trình có thể được thể hiện bởi một cặp `Value` và `Type`.     
`reflect.Type` và `reflect.Value` như tên gọi chứa các thông tin tương ứng về type và value của biến đang xem xét, đi kèm các utilitiy funtion và method để thao tác với data. Các giá trị này của một biến x bất kì có thể được lấy bằng `reflect.ValueOf(x)` và `reflect.TypeOf(x)`:
```go
type Person struct {
	Name string
	Age  int
}
p := Person{
	Name: "Duc",
	Age:  10,
}
pp := &p
o := []int{1, 2, 3}
s := "reflect in go"

fmt.Printf("(%v, %v)\n", reflect.ValueOf(p), reflect.TypeOf(p))
fmt.Printf("(%v, %v)\n", reflect.ValueOf(pp), reflect.TypeOf(pp))
fmt.Printf("(%v, %v)\n", reflect.ValueOf(o), reflect.TypeOf(o))
fmt.Printf("(%v, %v)\n", reflect.ValueOf(s), reflect.TypeOf(s))
```
Kết quả:

```
({Duc 10}, main.Person)
(&{Duc 10}, *main.Person)
([1 2 3], []int)
(reflect in go, string)
```

Ngoài ra ta cũng có thể đọc được thêm một số thông tin quan trọng khác như `reflect.Kind` - chứa thông tin cụ thể hơn về kiểu của một biến, có thể truy cập bằng `Value.Kind()` hoặc `Type.Kind()`:
```go
fmt.Printf("(%v, %v)\n", reflect.TypeOf(p), reflect.ValueOf(p).Kind())
fmt.Printf("(%v, %v)\n", reflect.TypeOf(pp), reflect.ValueOf(pp).Kind())
fmt.Printf("(%v, %v)\n", reflect.TypeOf(o), reflect.ValueOf(o).Kind())
fmt.Printf("(%v, %v)\n", reflect.TypeOf(s), reflect.ValueOf(s).Kind())
```

Kết quả:

```
(main.Person, struct)
(*main.Person, ptr)
([]int, slice)
(string, string)

```

## Parse url query với reflection

Xét bài toán cần viết một hàm nhận vào một `*http.Request` và một struct sau đó fill dữ liệu từ URL query vào struct đó:
```go
type Person struct {
	Name string
	Age  int
}

func ParseQuery(r *http.Request, p *Person) (err error) {
	q := r.URL.Query()
	p.Name = q.Get("name")
	p.Age, err = strconv.Atoi(q.Get("age"))
	return err
}

func TestParseQuery(t *testing.T) {
	req, _ := http.NewRequest("GET", "/root?name=Duc&age=10", nil)
	var d Person
	err := Parse(req, &d)
	if err != nil {
		t.Error(err.Error())
	}
	fmt.Println(d) // {Duc 10}
}
```
Một ví dụ khá cơ bản và thường gặp phải không? Tuy nhiên cách làm này sẽ cần phải lặp lại code khá nhiều.  
Vậy thay vì biết trước struct được truyền vào là `Person` thì ta có thể  truyền một struct bất kì với các field có tên ứng với các query có thể gặp mà vẫn đạt được kết quả tương tự hay không? Với reflect thì hoàn toàn có thể.  
Ý tưởng là có thể dùng reflect để đọc thông tin của struct bất kỳ được truyền vào, duyệt qua lần lượt các field, và kiểm tra xem trong URL có query nào tương ứng với field đang xét hay không, nếu có thì đọc giá trị của query vào field.  
### Bắt tay vào code
Tổng quan của chương trình có thể được thể hiện như sau:
```go
func Parse(r *http.Request, dest any) (err error) {
	v := reflect.ValueOf(dest)
	q := r.URL.Query()
	if v.Kind() != reflect.Ptr || v.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("dest must be a pointer to a struct")
	}
	v = v.Elem()
	t := v.Type()
	for i := 0; i < v.NumField(); i++ {
		fVal := v.Field(i)
		fType := t.Field(i)
		fName := strings.ToLower(fType.Name)
		err = parse(q.Get(fName), fVal)
		if err != nil {
			return fmt.Errorf("parse %w", err)
		}
	}
	return nil
}

func parse(stringVal string, destVal reflect.Value) (err error) {
    // xử lý cụ thể cho từng field
}
```

Lưu ý là bắt buộc đầu vào của hàm phải là một con trỏ tới struct thì reflect mới có thể thay đổi được struct đó, ta có thể kiểm tra điều kiện này với `v.Kind() == reflect.Ptr` và `v.Elem().Kind() == reflect.Struct` - Vì ta expect giá trị truyền vào là con trỏ (tương ứng `reflect.Ptr` nên cần gọi `Elem()` để lấy ra giá trị thực ở sau con trỏ đó)
Sau khi đã có được Type và Value của struct đầu vào rồi, ta sẽ tiến hành duyệt qua từng field và xử lý cụ thể ở trong hàm `parse`

```go
func parse(stringVal string, destVal reflect.Value) (err error) {
	if stringVal == "" { // bỏ qua nếu như không có query tương ứng với field này 
		return nil
	}
	if !destVal.CanSet() {
		return fmt.Errorf("field unexported or cannot set value")
	}
	k := destVal.Kind()
	switch {
	case k == reflect.String:
		err = parseString(stringVal, destVal)
	case k >= reflect.Int && k <= reflect.Int64:
		err = parseInt(stringVal, destVal)
	default:
		err = fmt.Errorf("type not supported: %v", destVal.Type())
	return err
}
```
Giải thích: Trong hàm này ta sẽ kiểm tra Kind của từng field và với mỗi kind đó sẽ có hàm cụ thể để xử lý giúp cho code clear hơn.
Trick nhỏ là thay vì kiểm tra với từng kiểu int, int8... thì có thể  viết `k >= reflect.Int && k <= reflect.Int64` do trong mã nguồn của reflect, các Kind có thể có của biến được viết dưới dạng:
```go
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	...
)
```
Tương tự với `uint` và `float`.
>Lưu ý quan trọng: reflect không thể cập nhật unexported field, nên cần kiểm tra trước với `CanSet()` (tương ứng với `CanAddr() == true` và field exported).
hoặc có thể kiểm tra với `CanInterface()`

Viết hàm parse đối với từng Kind:
```go
func parseString(in string, dest reflect.Value) error {
	dest.SetString(in)
	return nil
}

func parseInt(in string, dest reflect.Value) error {
	intVal, err := strconv.ParseInt(in, 10, 0)
	if err != nil {
		return fmt.Errorf("parseInt %w", err)
	}
	dest.SetInt(intVal)
	return nil
}
```

Tiến hành chạy thử chương trình:
```go
type Person struct {
	Name string
	Age  int
}

func TestQueryParser(t *testing.T) {
	req, _ := http.NewRequest("GET", "/root?name=Duc&age=10", nil)
	var d Person
	err := Parse(req, &d)
	if err != nil {
		t.Error(err.Error())
	}
	fmt.Println(d) // {Duc 10}
}
```
It works!
### Thế còn slice thì sao?
Đầu vào:
```go
type Person struct {
	...
	IDs []int
}
```
Expect với URL query có dạng `?ids=1,2,3`, sau khi parse thì field IDs sẽ có giá trị `[]int{1,2,3}`.  

Thêm case đối với `Kind == reflect.Slice` ở hàm `parse`
```go
func parse(stringVal string, destVal reflect.Value) (err error) {
	...
	case k == reflect.Slice:
		err = parseSlice(stringVal, destVal)
	...
}

func parseSlice(in string, dest reflect.Value) error {
	parts := strings.Split(in, ",")
	sliceType := dest.Type().Elem() // lấy type của phần tử strong slice
	sliceLen := len(parts)
	sliceVal := reflect.MakeSlice(reflect.SliceOf(sliceType), sliceLen, sliceLen) // make slice tương ứng
	for i := 0; i < sliceLen; i++ { // xử lý cho từng phần tử trong slice tương tự như struct ở trên
		err := parse(parts[i], sliceVal.Index(i)) // parse từng phần tử trong slice như đã làm với int và string
		if err != nil {
			return fmt.Errorf("parseSlice %w", err)
		}
	}
	dest.Set(sliceVal)
	return nil
}
```
Để chuyển từ URL query sang slice, ta cần kiểm tra xem ở struct đích slice có kiểu dữ liệu gì và make slice tương ứng, những công việc sau đó không khác gì so với xử lý struct ở phần trên.  
Chạy thử với slice:

```go
type Person struct {
	Name string
	Age  int
	IDs []int
}

func TestQueryParser(t *testing.T) {
	req, _ := http.NewRequest("GET", "/root?ids=1,2,3", nil)
	var d Person
	err := Parse(req, &d)
	if err != nil {
		t.Error(err.Error())
	}
	fmt.Println(d.IDs) // [1 2 3]
}
```

### Parse query linh động với struct tag
Ở phiên bản hiện tại, chương trình dựa trên tên của các field trong struct để từ đó lấy ra query tương ứng. Để chương trình được flexible hơn, có thể dùng struct tag để chỉ định query tương ứng với từng field.
Ví dụ
```go
type Person struct {
	Name string `query:"title"`
	...
}
```
Với sự hiện diện của tag `query`, field `Name` sẽ được parse từ query `title`, các field không có tag `query` thì behavior vẫn không thay đổi.  
Bổ sung thêm phần đọc struct tag cho hàm `Parse`:
```go
func Parse(r *http.Request, dest any) (err error) {
	v := reflect.ValueOf(dest)
	q := r.URL.Query()
	if !v.IsValid() || v.Kind() != reflect.Ptr || v.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("dest must be a pointer to not nil struct")
	}
	v = v.Elem()
	t := v.Type()
	for i := 0; i < v.NumField(); i++ {
		fVal := v.Field(i)
		fType := t.Field(i)
		fName := strings.ToLower(fType.Name)
		if queryTag := fType.Tag.Get("query"); queryTag != "" { // kiểm tra field có tag query hay không
			fName = queryTag
		}
		err = parse(q.Get(fName), fVal)
		if err != nil {
			return fmt.Errorf("parse %w", err)
		}
	}
	return nil
}
```
Và chạy thử:
```go
type Person struct {
	Name string `query:"title"`
	Age  int
	IDs  []int
}

func TestQueryParser(t *testing.T) {
	req, _ := http.NewRequest("GET", "/root?name=Duc&age=10&title=Gopher", nil)
	var d Person
	err := Parse(req, &d)
	if err != nil {
		t.Error(err.Error())
	}
	fmt.Println(d) // {Gopher 10 []}
}
```

### Custom parse với interface
Đến phiên bản hiện tại, parser đã có thể đạp ứng được các kiều dữ liệu cơ bản trong đa số trường hợp, nhưng chưa thể hoạt động được với các kiểu dữ liệu tự định nghĩa.
Ví dụ với field `Name`, thay vì đơn thuần là một string, có thể là struct dạng:
```go
type Name struct {
	First string
	Last  string
}

type Person struct {
	Name Name
	Age  int
	IDs  []int
}
```
Expect với URL query có dạng `?name=duc_hoang` thì sau khi parse query, giá trị của `Person.Name` sẽ là `Name{First:"duc",Last:"hoang"}`.  
Một lưu ý khi viết các hàm hay thư viện với reflect, thì nên hạn chế việc expose cho client phải thao tác với reflect để đơn giản hóa việc sử dụng hàm hay thư viện đó.
Trong trường hợp này, có thể dùng một interface để biểu thị kiểu dữ liệu tự định nghĩa có thể parse được, khi Parse và field đích implement interface này thì có thể dùng hàm tương ứng để xử lý field.
```go
type QueryParser interface {
	QueryParse(string) error
}
```

Implement QueryParser interface cho kiểu Name:
```go
func (n *Name) QueryParse(in string) error {
	parts := strings.Split(in, "_")
	if len(parts) != 2 {
		return fmt.Errorf("invalid input")
	}
	n.First = parts[0]
	n.Last = parts[1]
	return nil
}
```
Lưu ý là cần implement đối với pointer receiver để method có thể thay đổi giá trị của receiver.  
Tiến hành handle trong function parse:
```go
func parse(stringVal string, destVal reflect.Value) (err error) {
	...
	default:
		err = parseDefault(stringVal, destVal) // chuyển Kind mặc định ra handle riêng để đảm bảo code được clear và dễ maintain
	}
	return err
}

func parseDefault(in string, dest reflect.Value) error {
	if dest.Kind() != reflect.Ptr {
		dest = dest.Addr() // lấy con trỏ của dest nếu dest đang không phải con trỏ
	} else if dest.IsNil() {
		dest.Set(reflect.New(dest.Type().Elem())) // khởi taọ nếu dest là con trỏ nil
	}
	if queryParser, ok := dest.Interface().(QueryParser); ok {
		return queryParser.QueryParse(in) // parser giá trị từ query vào dest 
	}
	return fmt.Errorf("type not supported: %s", dest.Type().Kind())
}

```
Trong hàm `parseDefault`, giá trị đích sẽ luôn được chuyển sang kiểu con trỏ trước khi xác định xem nó có implement QueryParser hay không.  
Trong trường hợp giá trị đích đã là con trỏ rồi, cần kiểm tra xem có phải `nil` hay không, nếu là `nil` thì cần phải khởi tạo trước khi gọi method.  
Và chạy thử:
```go
type Name struct {
	First string
	Last  string
}

func (n *Name) QueryParse(in string) error {
	parts := strings.Split(in, "_")
	if len(parts) != 2 {
		return fmt.Errorf("invalid input")
	}
	n.First = parts[0]
	n.Last = parts[1]
	return nil
}

type Person struct {
	Name Name
	Age  int
	IDs  []int
}

func TestQueryParser(t *testing.T) {
	req, _ := http.NewRequest("GET", "/root?name=duc_hoang", nil)
	var d Person
	err := Parse(req, &d)
	if err != nil {
		t.Error(err.Error())
	}
	fmt.Printf("(%s, %s)", d.Name.First, d.Name.Last) // (duc, hoang)
}
```

Source code trong bài: https://github.com/duchng/queryparser



