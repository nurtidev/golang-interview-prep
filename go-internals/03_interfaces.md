# Интерфейсы (Interfaces)

---

## Лекция

### Что такое интерфейс и зачем он нужен

Интерфейс — это **контракт**: набор методов, которые тип должен реализовать. Интерфейс описывает **поведение**, а не данные.

Зачем? Чтобы писать код, который работает с любым типом, реализующим нужные методы — без привязки к конкретной реализации. Классический пример: `io.Writer` принимает `*os.File`, `*bytes.Buffer`, `*net.Conn` — любой тип с методом `Write([]byte) (int, error)`.

Это основа **полиморфизма** в Go.

---

### Неявная реализация — ключевое отличие от Java/C#

В Java/C# нужно явно написать `implements Interface`. В Go — нет. Тип автоматически реализует интерфейс, если у него есть все нужные методы.

```go
type Stringer interface {
    String() string
}

type User struct{ Name string }

// User нигде не объявляет "я реализую Stringer"
// Но метод String() есть → User автоматически реализует Stringer
func (u User) String() string {
    return u.Name
}

var s Stringer = User{Name: "Nurtilek"} // ✅ работает
```

Преимущество: можно реализовать интерфейс из **чужого пакета** для своего типа, и наоборот — определить интерфейс под существующий тип не меняя его код.

---

### Внутреннее устройство

Интерфейс — это структура из **двух указателей**: тип и данные.

```go
// Непустой интерфейс (iface) — для interface{ methods... }
type iface struct {
    tab  *itab          // указатель на itab: тип + таблица методов
    data unsafe.Pointer // указатель на конкретные данные
}

type itab struct {
    inter *interfacetype // тип интерфейса (какие методы требуются)
    _type *_type         // конкретный тип (кто реализует)
    fun   [...]uintptr   // vtable: адреса методов конкретного типа
}

// Пустой интерфейс (eface) — для any / interface{}
type eface struct {
    _type *_type         // конкретный тип
    data  unsafe.Pointer // данные
}
```

Визуально:

```
var w io.Writer = &bytes.Buffer{}

w (iface):
┌─────────────┐
│ tab  ────────────→ itab { inter=io.Writer, type=*bytes.Buffer, fun=[Write→...] }
│ data ────────────→ bytes.Buffer{...}
└─────────────┘
```

**Что происходит при вызове метода через интерфейс:**
1. Берём `tab.fun[i]` — адрес метода из vtable
2. Вызываем функцию, передавая `data` как первый аргумент (ресивер)

Это **косвенный вызов** — чуть медленнее прямого. Компилятор не может инлайнить такой вызов без escape analysis.

---

### Главная ловушка: nil interface

Это самый популярный вопрос на собеседовании по Go-интерфейсам.

```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func getError() error {
    var err *MyError = nil // typed nil pointer
    return err             // ← ВОТ ПРОБЛЕМА
}

func main() {
    err := getError()
    if err != nil {
        fmt.Println("got error!") // ← это выполнится, хотя "ошибки" нет
    }
}
```

**Почему?**

```
возвращаем typed nil:      интерфейс error:
*MyError(nil)    →    iface{ tab: *(*MyError)метод-таблица,  data: nil }
                             ↑
                      tab НЕ nil → интерфейс НЕ nil!
```

Интерфейс равен `nil` только когда **оба поля nil**: `iface{tab: nil, data: nil}`. Если тип известен (tab != nil), интерфейс не nil — даже если сам указатель nil.

**Правило:** если функция возвращает интерфейс и "нет ошибки" — возвращай `nil` напрямую, не через типизированную переменную:

```go
// ❌ Проблема
func getError() error {
    var err *MyError = nil
    return err // iface{tab:*MyError, data:nil} — НЕ nil интерфейс
}

// ✅ Правильно
func getError() error {
    return nil // eface{nil, nil} — настоящий nil интерфейс
}

// ✅ Тоже правильно — явная проверка
func getError() error {
    var err *MyError = nil
    if err == nil {
        return nil // возвращаем nil напрямую
    }
    return err
}
```

---

### Сравнение интерфейсов

Два интерфейса равны когда **оба поля** (тип И данные) совпадают:

```go
var a, b error

a = errors.New("oops")
b = errors.New("oops")
fmt.Println(a == b) // false — разные указатели (data разные)

var c, d error
fmt.Println(c == d) // true — оба nil (iface{nil, nil})
```

**Паника при сравнении несравниваемого конкретного типа:**
```go
var x interface{} = []int{1, 2, 3}
var y interface{} = []int{1, 2, 3}
fmt.Println(x == y) // ПАНИКА: comparing uncomparable type []int
```

Компилятор не может это проверить (тип хранится в runtime), поэтому паника только в runtime.

---

### Type Assertion — приведение типа

Извлекает конкретный тип из интерфейса:

```go
var i interface{} = "hello"

// Небезопасный: паника если тип не совпадает
s := i.(string) // ✅ работает
n := i.(int)    // panic: interface conversion: interface {} is string, not int

// Безопасный: возвращает ok
s, ok := i.(string) // ok=true, s="hello"
n, ok := i.(int)    // ok=false, n=0 (zero value)
if !ok {
    fmt.Println("не int")
}
```

**Правило:** всегда используй безопасную форму `v, ok := i.(T)` если не уверен в типе.

---

### Type Switch — выбор по типу

```go
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case []int:
        return fmt.Sprintf("[]int: len=%d", len(v))
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("неизвестный тип: %T", v)
    }
}
```

В каждом case переменная `v` имеет тип этого case — не интерфейс. Это очень удобно.

---

### Embedding интерфейсов

Интерфейсы можно встраивать — создавать составные интерфейсы:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter = Reader + Writer
type ReadWriter interface {
    Reader
    Writer
}

// *os.File реализует оба → автоматически реализует ReadWriter
var rw ReadWriter = os.Stdin
```

Стандартная библиотека активно использует этот паттерн: `io.ReadWriter`, `io.ReadWriteCloser`, `io.ReadWriteSeeker`.

---

### Пустой интерфейс: any / interface{}

`any` — алиас для `interface{}` (с Go 1.18). Может хранить значение любого типа.

```go
var x any
x = 42
x = "hello"
x = []int{1, 2, 3}
x = struct{ Name string }{"Go"}
```

Используй `any` когда тип действительно неизвестен заранее (обобщённые контейнеры, JSON-парсинг, логгирование). В остальных случаях — предпочитай конкретные типы или generics.

---

### Обработка ошибок: errors.Is и errors.As

Go 1.13 добавил обёртку ошибок через `%w` и функции для работы с цепочками ошибок:

```go
type NotFoundError struct{ ID int }
func (e *NotFoundError) Error() string { return fmt.Sprintf("not found: %d", e.ID) }

// Оборачиваем ошибку — сохраняем исходную в цепочке
wrapped := fmt.Errorf("db query failed: %w", &NotFoundError{ID: 42})

// errors.Is: проверяет цепочку на конкретное значение (через ==)
if errors.Is(wrapped, ErrNotFound) { ... }

// errors.As: ищет в цепочке тип и заполняет target
var nfe *NotFoundError
if errors.As(wrapped, &nfe) {
    fmt.Println("ID:", nfe.ID) // 42
}
```

**Разница:**
- `errors.Is` — для sentinel errors (конкретные значения: `io.EOF`, `sql.ErrNoRows`)
- `errors.As` — для типов ошибок с дополнительными данными

---

### Производительность интерфейсов

**Косвенный вызов метода:** чуть медленнее прямого из-за разыменования vtable. На практике незначительно — наносекунды. Важно только в самых горячих inner loops.

**Боксинг (allocation при присваивании в интерфейс):** когда помещаем значение в интерфейс — оно может быть скопировано в heap если его размер > machine word. Это аллокация.

```go
// Аллокация может быть здесь:
var w io.Writer = &bytes.Buffer{} // указатель — без аллокации (ptr fits in data field)
var i interface{} = 42            // int обычно не аллоцируется (оптимизация)
var i interface{} = [100]byte{...} // большой массив — аллокация
```

Правило: не оборачивай в интерфейс в критичных по производительности местах если тип известен статически.

---

## Примеры

### Пример 1: Полиморфизм через интерфейсы

```go
package main

import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
    Perimeter() float64
}

type Circle struct{ Radius float64 }
type Rectangle struct{ Width, Height float64 }

func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

func printShapeInfo(s Shape) {
    fmt.Printf("%T: area=%.2f, perimeter=%.2f\n", s, s.Area(), s.Perimeter())
}

func main() {
    shapes := []Shape{
        Circle{Radius: 5},
        Rectangle{Width: 4, Height: 3},
    }

    for _, s := range shapes {
        printShapeInfo(s)
    }
}
```

### Пример 2: Nil interface trap — демонстрация и исправление

```go
package main

import "fmt"

type AppError struct{ Code int; Msg string }
func (e *AppError) Error() string { return fmt.Sprintf("[%d] %s", e.Code, e.Msg) }

// ❌ Ловушка: возвращаем typed nil
func badValidate(input string) error {
    var err *AppError
    if input == "" {
        err = &AppError{Code: 400, Msg: "empty input"}
    }
    return err // typed nil → интерфейс НЕ nil даже когда err==nil!
}

// ✅ Правильно: явно возвращаем nil
func goodValidate(input string) error {
    if input == "" {
        return &AppError{Code: 400, Msg: "empty input"}
    }
    return nil // настоящий nil интерфейс
}

func main() {
    err := badValidate("hello")
    if err != nil {
        fmt.Println("badValidate: ошибка есть (НО НЕ ДОЛЖНА):", err)
    }

    err = goodValidate("hello")
    if err != nil {
        fmt.Println("goodValidate: ошибка есть")
    } else {
        fmt.Println("goodValidate: всё хорошо") // ← выведется это
    }
}
```

### Пример 3: Type Switch для обработки разных типов событий

```go
package main

import "fmt"

type LoginEvent struct{ UserID string }
type LogoutEvent struct{ UserID string }
type ErrorEvent struct{ Code int; Msg string }

func handleEvent(event interface{}) {
    switch e := event.(type) {
    case LoginEvent:
        fmt.Printf("пользователь %s вошёл\n", e.UserID)
    case LogoutEvent:
        fmt.Printf("пользователь %s вышел\n", e.UserID)
    case ErrorEvent:
        fmt.Printf("ошибка %d: %s\n", e.Code, e.Msg)
    default:
        fmt.Printf("неизвестное событие: %T\n", e)
    }
}

func main() {
    events := []interface{}{
        LoginEvent{UserID: "alice"},
        ErrorEvent{Code: 500, Msg: "internal error"},
        LogoutEvent{UserID: "alice"},
    }

    for _, e := range events {
        handleEvent(e)
    }
}
```

### Пример 4: Dependency Injection через интерфейсы

```go
package main

import (
    "fmt"
)

// Интерфейс — зависимость, а не конкретная реализация
type UserRepository interface {
    FindByID(id int) (string, error)
    Save(id int, name string) error
}

type UserService struct {
    repo UserRepository // зависит от интерфейса, не от конкретной БД
}

func (s *UserService) GetUser(id int) string {
    name, err := s.repo.FindByID(id)
    if err != nil {
        return "unknown"
    }
    return name
}

// Реальная реализация
type InMemoryRepo struct {
    data map[int]string
}

func (r *InMemoryRepo) FindByID(id int) (string, error) {
    name, ok := r.data[id]
    if !ok {
        return "", fmt.Errorf("user %d not found", id)
    }
    return name, nil
}

func (r *InMemoryRepo) Save(id int, name string) error {
    r.data[id] = name
    return nil
}

// Мок для тестов
type MockRepo struct{ ReturnName string }
func (m *MockRepo) FindByID(id int) (string, error) { return m.ReturnName, nil }
func (m *MockRepo) Save(id int, name string) error  { return nil }

func main() {
    // Продакшн
    repo := &InMemoryRepo{data: map[int]string{1: "Alice", 2: "Bob"}}
    service := &UserService{repo: repo}
    fmt.Println(service.GetUser(1)) // Alice

    // Тест — подменяем реализацию
    mockService := &UserService{repo: &MockRepo{ReturnName: "TestUser"}}
    fmt.Println(mockService.GetUser(999)) // TestUser
}
```

### Пример 5: errors.Is и errors.As с цепочкой ошибок

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

type DBError struct {
    Table string
    Err   error
}

func (e *DBError) Error() string { return fmt.Sprintf("db(%s): %v", e.Table, e.Err) }
func (e *DBError) Unwrap() error { return e.Err } // для errors.Is/As

func queryUser(id int) error {
    if id == 0 {
        return &DBError{
            Table: "users",
            Err:   fmt.Errorf("query failed: %w", ErrNotFound),
        }
    }
    return nil
}

func main() {
    err := queryUser(0)

    // errors.Is проходит по цепочке: DBError → fmt.Errorf wrapper → ErrNotFound
    if errors.Is(err, ErrNotFound) {
        fmt.Println("пользователь не найден") // ← выведется
    }

    // errors.As достаёт конкретный тип из цепочки
    var dbErr *DBError
    if errors.As(err, &dbErr) {
        fmt.Printf("ошибка в таблице: %s\n", dbErr.Table) // users
    }
}
```

---

## Вопросы с собеседования

**Q: Что выведет этот код и почему?**
```go
var buf *bytes.Buffer
var w io.Writer = buf
fmt.Println(w == nil)
```

`false`. Хотя `buf == nil`, переменная `w` (тип `io.Writer`) содержит `iface{tab: *bytes.Buffer-vtable, data: nil}`. Поле `tab` не nil — интерфейс не nil. Интерфейс nil только когда оба поля nil: `iface{nil, nil}`.

---

**Q: Объясните nil interface trap.**

Если функция возвращает `error`, и мы присваиваем `nil` в типизированную переменную перед возвратом — возвращается "не-nil" интерфейс. Правило: всегда возвращай `nil` напрямую, не через промежуточную переменную типа `*ConcreteError`.

---

**Q: Чем `any` отличается от `interface{}`?**

Ничем — `any` это алиас для `interface{}` начиная с Go 1.18. Полностью взаимозаменяемы. `any` предпочтительнее в новом коде — короче и читабельнее.

---

**Q: Как работает type assertion? Что произойдёт если тип не совпадает?**

`v := i.(T)` — паника если `i` не содержит тип `T`. `v, ok := i.(T)` — безопасно: если тип не совпадает, `ok=false`, `v=zero value`, паники нет. Используй двойную форму если не уверен в типе.

---

**Q: Можно ли хранить интерфейс в map как ключ?**

Да, компилируется. Но паникует в runtime если конкретный тип, хранящийся в интерфейсе, не comparable (например, слайс или map). Компилятор не может это проверить статически.

---

**Q: В чём разница между errors.Is и errors.As?**

`errors.Is(err, target)` — проверяет, есть ли в цепочке ошибка равная `target` (сравнение по значению). Используется для sentinel errors: `io.EOF`, `sql.ErrNoRows`.

`errors.As(err, &target)` — ищет в цепочке ошибку указанного типа и заполняет `target`. Используется когда нужно извлечь данные из конкретного типа ошибки.

---

**Q: Почему Go использует неявную реализацию интерфейсов?**

Это даёт **loose coupling**: можно определить интерфейс под уже существующий тип, не меняя его код. И наоборот — сторонний тип автоматически удовлетворяет нашему интерфейсу если у него есть нужные методы. В Java нужно было бы менять оба класса.

---

**Q: Что такое vtable и как она используется при вызове метода через интерфейс?**

`itab` содержит массив указателей на функции (`fun [...]uintptr`) — по одному на каждый метод интерфейса. При вызове `w.Write(data)`: берём `w.tab.fun[0]` (адрес `Write`) и вызываем, передавая `w.data` как ресивер. Это **косвенный вызов** — в отличие от прямого вызова метода, компилятор не знает адрес заранее.

---

**Q: Когда вызов через интерфейс может быть проблемой по производительности?**

В самых горячих inner loops: косвенный вызов нельзя инлайнить (в общем случае), что мешает оптимизациям компилятора. Также возможна аллокация при боксинге большого значения в интерфейс. Решение: профилировать перед оптимизацией; в критичных местах использовать конкретные типы или generics.
