# Defer, Panic, Recover

---

## Лекция

### Что такое defer и зачем он нужен

Когда работаешь с ресурсами — файлами, соединениями, мьютексами — нужно их освобождать. Проблема: функция может иметь несколько путей выхода (`return`, ранний выход при ошибке). Без специального механизма придётся вставлять `Close()` / `Unlock()` в каждом `return`.

`defer` решает это: откладывает вызов функции до момента **возврата из окружающей функции**, независимо от того, как именно происходит этот возврат — через `return`, через `panic`, или через конец функции.

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close() // выполнится при любом return ниже

// ... работаем с файлом ...
// не нужно помнить про f.Close() в каждой ветке
```

---

### Порядок выполнения: LIFO (стек)

defer-ы накапливаются как стек. Последний `defer` выполнится первым:

```go
func main() {
    defer fmt.Println("первый defer → выполнится третьим")
    defer fmt.Println("второй defer → выполнится вторым")
    defer fmt.Println("третий defer → выполнится первым")
    fmt.Println("основной код")
}
// Вывод:
// основной код
// третий defer → выполнится первым
// второй defer → выполнится вторым
// первый defer → выполнится третьим
```

Почему LIFO? Это логично для парных операций: если открыли A, потом B — закрывать нужно сначала B, потом A.

```go
mu1.Lock()
defer mu1.Unlock()  // разблокируется вторым

mu2.Lock()
defer mu2.Unlock()  // разблокируется первым (LIFO)
```

---

### Аргументы вычисляются немедленно, не при выполнении

Это тонкий момент, который часто спрашивают на собеседованиях:

```go
func main() {
    x := 10
    defer fmt.Println(x) // x=10 захватывается ПРЯМО СЕЙЧАС, в момент defer
    x = 20
    fmt.Println(x)       // 20
}
// Вывод:
// 20
// 10  ← не 20! аргумент был вычислен при вызове defer
```

**Но:** если `defer` использует **замыкание** (func literal без аргументов) — переменная захватывается по ссылке и читается в момент реального выполнения:

```go
func main() {
    x := 10
    defer func() {
        fmt.Println(x) // читает x в момент выполнения defer
    }()
    x = 20
    fmt.Println(x)
}
// Вывод:
// 20
// 20  ← замыкание видит актуальное x=20
```

Правило: аргументы в `defer f(arg)` — вычисляются сразу. Тело `defer func() { ... }()` — выполняется позже.

---

### Именованные возвращаемые значения и defer

Именованные возвращаемые значения — переменные, объявленные прямо в сигнатуре. `defer` может их модифицировать:

```go
func double(x int) (result int) { // result — именованная переменная
    defer func() {
        result *= 2 // модифицируем result перед реальным возвратом
    }()
    result = x
    return // return без значения = "return result"
}

fmt.Println(double(5)) // 10, не 5
```

Это работает потому что `return` сначала присваивает значение именованной переменной, а потом выполняются defer-ы — которые могут её изменить.

**Важно: не именованный return**
```go
func f() int {
    x := 5
    defer func() { x++ }() // x++ не влияет на возвращаемое значение
    return x               // return x = "скопировали x в return value, вернули"
                           // defer меняет x, но уже скопированное значение не изменится
}
fmt.Println(f()) // 5, а не 6
```

---

### Антипаттерн: defer в цикле

`defer` выполняется при **выходе из функции**, а не из итерации цикла. Если defer внутри цикла — все вызовы накопятся до конца функции:

```go
// ❌ Плохо: все Close() выполнятся только в конце функции
// При большом количестве файлов — исчерпание файловых дескрипторов
func processFiles(paths []string) {
    for _, path := range paths {
        f, _ := os.Open(path)
        defer f.Close() // накапливается! не закрывается после каждой итерации
        process(f)
    }
    // ← только здесь все f.Close() выполнятся
}

// ✅ Правильно: обернуть в функцию — defer выполнится при выходе из неё
func processFiles(paths []string) {
    for _, path := range paths {
        func() {
            f, _ := os.Open(path)
            defer f.Close() // выполнится при выходе из анонимной функции
            process(f)
        }()
    }
}

// ✅ Или явный Close без defer (для простых случаев)
func processFiles(paths []string) {
    for _, path := range paths {
        f, _ := os.Open(path)
        process(f)
        f.Close() // явно, без defer
    }
}
```

---

### Производительность defer

**До Go 1.14:** `defer` был медленным — требовал heap allocation (~100 нс overhead). Это создавало проблемы в горячих путях.

**С Go 1.14:** введён "open-coded defer" (inline defer). Компилятор статически вставляет вызов defer-функции прямо в код функции — overhead почти нулевой. Heap allocation только если defer внутри цикла или `goto`.

Вывод: не бойся `defer` из соображений производительности в современном Go.

---

### Panic — экстренная остановка

`panic` прерывает нормальный поток выполнения:
1. Текущая функция останавливается
2. Начинается **раскрутка стека** (stack unwinding): Go поднимается по стеку вызовов
3. На каждом уровне выполняются все `defer`-ы
4. Программа падает с сообщением и stack trace

```go
func main() {
    defer fmt.Println("defer выполнится даже при панике")
    fmt.Println("до паники")
    panic("что-то пошло не так")
    fmt.Println("этот код не выполнится") // unreachable
}
// Вывод:
// до паники
// defer выполнится даже при панике
// goroutine 1 [running]:
// main.main()
//     panic: что-то пошло не так
```

**Когда Go сам вызывает panic:**
- Выход за границы слайса/массива: `index out of range`
- Разыменование nil-указателя: `nil pointer dereference`
- Запись в закрытый канал: `send on closed channel`
- Небезопасный type assertion: `interface conversion`
- Целочисленное деление на ноль: `integer divide by zero`
- Stack overflow: бесконечная рекурсия
- Конкурентная запись в map: `concurrent map writes`

**Когда использовать panic самостоятельно:**
- Ошибка программиста, которую нельзя обработать (невалидный аргумент в публичной функции)
- Инициализация пакета при недопустимых условиях (нет нужной переменной окружения)
- Никогда для обычных ошибок — для них возвращай `error`

---

### Recover — перехват паники

`recover()` останавливает раскрутку стека и возвращает значение, переданное в `panic`. **Работает только внутри `defer`:**

```go
func safeOperation() (err error) {
    defer func() {
        if r := recover(); r != nil {
            // превращаем панику в ошибку
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    // ... опасный код ...
    return nil
}
```

Если `recover()` вызвать не из defer (или если паники не было) — возвращает `nil`.

```go
// ❌ Не работает: recover не в defer
func badRecover() {
    if r := recover(); r != nil { // r будет nil, паника не поймана
        fmt.Println("recovered:", r)
    }
}
```

---

### Паника не передаётся между горутинами

Это критически важное ограничение:

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered in main:", r) // НЕ сработает
        }
    }()

    go func() {
        panic("паника в горутине") // убивает всю программу, не ловится в main
    }()

    time.Sleep(time.Second)
}
// Программа упадёт: goroutine 5 [running]: panic: паника в горутине
```

Каждая горутина должна сама обрабатывать свои паники через `defer + recover`. Паника в горутине без recover — это **crash всей программы**.

---

### Паттерн: безопасный запуск горутин

В production-коде горутины оборачивают в helper:

```go
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                // логируем, не роняем сервис
                log.Printf("goroutine panic: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}

// Использование:
safeGo(func() {
    processRequest(req) // если запаникует — поймаем выше
})
```

---

### runtime.Goexit() — мягкое завершение горутины

Отличается от panic: завершает текущую горутину, выполняя все defer-ы, но не является паникой — `recover()` не поймает.

```go
go func() {
    defer fmt.Println("defer выполнится")
    defer func() {
        r := recover() // r == nil — Goexit не паника
        fmt.Println("recover:", r)
    }()
    runtime.Goexit() // мягко завершаем горутину
    fmt.Println("этот код не выполнится")
}()
```

Используется в тестах: `t.FailNow()` и `t.Fatal()` внутри вызывают `runtime.Goexit()`.

---

## Примеры

### Пример 1: defer для очистки ресурсов

```go
package main

import (
    "fmt"
    "os"
)

func copyFile(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("open src: %w", err)
    }
    defer in.Close() // гарантированно закроется

    out, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("create dst: %w", err)
    }
    defer out.Close() // гарантированно закроется

    // ... копирование данных ...
    // не нужно думать про Close() в каждой ветке return
    return nil
}

func main() {
    // Создаём тестовый файл
    os.WriteFile("src.txt", []byte("hello"), 0644)
    defer os.Remove("src.txt") // удалим после теста
    defer os.Remove("dst.txt")

    err := copyFile("src.txt", "dst.txt")
    fmt.Println("error:", err)
}
```

### Пример 2: defer для измерения времени

```go
package main

import (
    "fmt"
    "time"
)

func timeTrack(name string) func() {
    start := time.Now()
    return func() {
        fmt.Printf("%s: %v\n", name, time.Since(start))
    }
}

func expensiveOperation() {
    defer timeTrack("expensiveOperation")() // вызов сразу, возвращает func
    // сокращение: аргументы defer вычисляются сразу (start захватывается здесь)
    // возвращённая func() выполнится при return

    time.Sleep(100 * time.Millisecond)
    // ... тяжёлая работа ...
}

func main() {
    expensiveOperation()
    // expensiveOperation: 100.xxx ms
}
```

### Пример 3: recover — превращаем панику в ошибку

```go
package main

import "fmt"

// safeDiv безопасно делит, превращая панику в ошибку
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("division failed: %v", r)
        }
    }()
    return a / b, nil
}

// safeIndex безопасно обращается к элементу слайса
func safeIndex(s []int, i int) (val int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("index out of range: %d (len=%d)", i, len(s))
        }
    }()
    return s[i], nil
}

func main() {
    result, err := safeDiv(10, 0)
    fmt.Printf("10/0: result=%d, err=%v\n", result, err)

    result, err = safeDiv(10, 2)
    fmt.Printf("10/2: result=%d, err=%v\n", result, err)

    val, err := safeIndex([]int{1, 2, 3}, 10)
    fmt.Printf("index 10: val=%d, err=%v\n", val, err)
}
```

### Пример 4: Именованный return + defer для логирования

```go
package main

import (
    "fmt"
    "errors"
)

var ErrNotFound = errors.New("not found")

func findUser(id int) (name string, err error) {
    // Логируем результат при выходе — видим и name и err
    defer func() {
        if err != nil {
            fmt.Printf("findUser(%d): error: %v\n", id, err)
        } else {
            fmt.Printf("findUser(%d): found: %s\n", id, name)
        }
    }()

    users := map[int]string{1: "Alice", 2: "Bob"}
    name, ok := users[id]
    if !ok {
        err = fmt.Errorf("user %d: %w", id, ErrNotFound)
        return // именованный return: err уже установлен
    }
    return // name установлен, err == nil
}

func main() {
    findUser(1)  // findUser(1): found: Alice
    findUser(99) // findUser(99): error: user 99: not found
}
```

### Пример 5: Безопасный HTTP-хэндлер с recover

```go
package main

import (
    "fmt"
    "log"
    "runtime/debug"
)

type Handler func(req string) string

// Оборачиваем хэндлер для защиты от паник
func safeHandler(h Handler) Handler {
    return func(req string) (result string) {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("handler panic for req=%q: %v\n%s", req, r, debug.Stack())
                result = "500 Internal Server Error"
            }
        }()
        return h(req)
    }
}

func riskyHandler(req string) string {
    if req == "bad" {
        panic("unexpected input")
    }
    return fmt.Sprintf("200 OK: processed %q", req)
}

func main() {
    handler := safeHandler(riskyHandler)

    fmt.Println(handler("hello"))  // 200 OK: processed "hello"
    fmt.Println(handler("bad"))    // 500 Internal Server Error (паника поймана)
    fmt.Println(handler("world"))  // 200 OK: processed "world"
}
```

---

## Вопросы с собеседования

**Q: Что выведет?**
```go
func main() {
    defer fmt.Println("A")
    defer fmt.Println("B")
    defer fmt.Println("C")
    fmt.Println("main")
}
```

`main`, `C`, `B`, `A`. Порядок LIFO: последний defer — первым.

---

**Q: Что выведет?**
```go
func f() int {
    x := 5
    defer func() { x++ }()
    return x
}
fmt.Println(f())
```

`5`. `return x` — это "скопировать x в return value". Defer меняет `x`, но не return value (неименованный return). Если бы был `func f() (x int)` — вывело бы `6`.

---

**Q: Что выведет?**
```go
func main() {
    x := 10
    defer fmt.Println(x)
    x = 20
}
```

`10`. Аргументы defer вычисляются **в момент вызова defer** — `x=10` захватывается сразу. Изменение `x=20` не влияет на уже захваченное значение.

---

**Q: В чём разница между `defer fmt.Println(x)` и `defer func() { fmt.Println(x) }()`?**

В первом случае `x` вычисляется немедленно при регистрации defer — захватывается текущее значение. Во втором — замыкание захватывает переменную по ссылке, значение читается при реальном выполнении defer (при выходе из функции). Второй вариант увидит актуальное значение `x` на момент return.

---

**Q: Почему нельзя использовать defer в цикле для закрытия файлов?**

`defer` выполняется при выходе из **функции**, а не из итерации цикла. Все defer-ы накапливаются и выполняются только в конце функции. При большом количестве файлов — все файловые дескрипторы будут открыты одновременно до конца функции. Решение: обернуть тело цикла в анонимную функцию.

---

**Q: Когда следует использовать panic?**

- Программная ошибка, которую нельзя обработать в рантайме (нарушение инварианта, невалидный конфиг)
- В `init()` когда инициализация невозможна без нужных ресурсов
- Никогда для обычных ошибок — для них `error`. `panic` — это исключение, а не поток управления.

---

**Q: Может ли recover() перехватить панику из другой горутины?**

Нет. `recover()` перехватывает только паники текущей горутины. Паника в другой горутине без собственного recover — это краш всей программы. Каждая горутина должна сама обрабатывать свои паники через `defer + recover`.

---

**Q: Что вернёт recover() если паники не было?**

`nil`. Проверка `if r := recover(); r != nil` корректно обрабатывает оба случая.

---

**Q: Чем `runtime.Goexit()` отличается от `panic`?**

`panic` — аварийное завершение с раскруткой стека; `recover()` может поймать. `runtime.Goexit()` — корректное завершение текущей горутины; `recover()` не поймает (вернёт nil). Оба выполняют все defer-ы. `Goexit` используется в тестах (`t.Fatal`) и не считается ошибкой.

---

**Q: Как правильно логировать ошибку на выходе из функции с именованным return?**

```go
func doWork() (err error) {
    defer func() {
        if err != nil {
            log.Printf("doWork failed: %v", err)
        }
    }()
    // ... работа ...
}
```

Defer видит актуальное значение `err` в момент выполнения (именованный return), поэтому логирует реальную ошибку.
