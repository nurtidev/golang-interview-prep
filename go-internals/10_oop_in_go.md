# ООП в Go

---

## Лекция

### Есть ли ООП в Go?

Формально Go **не является ООП-языком** в классическом смысле — нет классов, нет наследования, нет `abstract`. Но Go поддерживает **три из четырёх** столпов ООП:

| Принцип ООП      | Go              | Как реализовано                        |
|------------------|-----------------|----------------------------------------|
| Инкапсуляция     | ✅ есть          | unexported поля/методы (строчная буква)|
| Полиморфизм      | ✅ есть          | интерфейсы (неявная реализация)        |
| Абстракция       | ✅ есть          | интерфейсы как контракты               |
| Наследование     | ❌ нет           | заменяется **embedding** (композицией) |

Философия Go: **"композиция лучше наследования"**.

---

### 1. Инкапсуляция

В Go инкапсуляция реализована через **видимость на уровне пакета**:
- Имя с **заглавной буквы** → экспортируемое (публичное)
- Имя со **строчной буквы** → неэкспортируемое (приватное)

```go
package account

type BankAccount struct {
    owner   string  // приватное — снаружи пакета не видно
    balance float64 // приватное
    ID      int     // публичное — видно везде
}

// Конструктор — единственный способ создать с валидацией
func NewBankAccount(owner string) *BankAccount {
    return &BankAccount{owner: owner, balance: 0}
}

// Публичные методы — контролируемый доступ
func (a *BankAccount) Deposit(amount float64) error {
    if amount <= 0 {
        return errors.New("amount must be positive")
    }
    a.balance += amount
    return nil
}

func (a *BankAccount) Balance() float64 {
    return a.balance // только чтение снаружи
}

// Приватный метод — только внутри пакета
func (a *BankAccount) validate() bool {
    return a.balance >= 0
}
```

```go
package main

import "account"

func main() {
    acc := account.NewBankAccount("Nurtilek")
    acc.Deposit(1000)
    fmt.Println(acc.Balance()) // ✅ 1000

    // acc.balance = -999  // ❌ compile error: unexported field
    // acc.validate()      // ❌ compile error: unexported method
}
```

---

### 2. Полиморфизм через интерфейсы

В Go нет `virtual` методов и нет `override`. Вместо этого — **утиная типизация**: если тип реализует все методы интерфейса, он автоматически им является.

```go
// Интерфейс = абстрактный контракт
type Animal interface {
    Sound() string
    Name() string
}

type Dog struct{ name string }
type Cat struct{ name string }
type Duck struct{ name string }

func (d Dog) Sound() string  { return "Woof" }
func (d Dog) Name() string   { return d.name }

func (c Cat) Sound() string  { return "Meow" }
func (c Cat) Name() string   { return c.name }

func (d Duck) Sound() string { return "Quack" }
func (d Duck) Name() string  { return d.name }

// Полиморфная функция — работает с любым Animal
func MakeSound(a Animal) {
    fmt.Printf("%s says: %s\n", a.Name(), a.Sound())
}

func main() {
    animals := []Animal{
        Dog{name: "Rex"},
        Cat{name: "Whiskers"},
        Duck{name: "Donald"},
    }

    for _, a := range animals {
        MakeSound(a) // полиморфизм в действии
    }
}
// Rex says: Woof
// Whiskers says: Meow
// Donald says: Quack
```

**Ключевое отличие от Java:** `Dog` нигде не пишет `implements Animal`. Это позволяет реализовывать интерфейсы из чужих пакетов.

---

### 3. Абстракция

Интерфейсы скрывают детали реализации — это и есть абстракция.

```go
// Абстракция хранилища — неважно, где хранятся данные
type UserRepository interface {
    FindByID(id int) (*User, error)
    Save(user *User) error
    Delete(id int) error
}

// Реализация 1: PostgreSQL
type PostgresUserRepo struct {
    db *sql.DB
}
func (r *PostgresUserRepo) FindByID(id int) (*User, error) { /* SQL запрос */ }
func (r *PostgresUserRepo) Save(user *User) error          { /* SQL INSERT */ }
func (r *PostgresUserRepo) Delete(id int) error            { /* SQL DELETE */ }

// Реализация 2: In-memory (для тестов)
type InMemoryUserRepo struct {
    data map[int]*User
}
func (r *InMemoryUserRepo) FindByID(id int) (*User, error) { return r.data[id], nil }
func (r *InMemoryUserRepo) Save(user *User) error          { r.data[user.ID] = user; return nil }
func (r *InMemoryUserRepo) Delete(id int) error            { delete(r.data, id); return nil }

// Сервис работает через абстракцию — не знает о конкретной реализации
type UserService struct {
    repo UserRepository // инъекция зависимости
}

func (s *UserService) GetUser(id int) (*User, error) {
    return s.repo.FindByID(id)
}
```

---

### 4. "Наследование" через Embedding (композиция)

В Go нет наследования — вместо него **embedding** (встраивание). Это **не наследование**, а делегирование: встроенный тип остаётся независимым.

```go
// Базовый "класс"
type Animal struct {
    Name string
    Age  int
}

func (a Animal) Describe() string {
    return fmt.Sprintf("%s, age %d", a.Name, a.Age)
}

func (a Animal) Breathe() string {
    return "breathing..."
}

// Dog "наследует" Animal через embedding
type Dog struct {
    Animal           // встроенный тип — поля/методы доступны напрямую
    Breed  string
}

// "Переопределение" метода — просто новый метод на Dog
func (d Dog) Describe() string {
    return fmt.Sprintf("%s (%s breed)", d.Animal.Describe(), d.Breed)
}

func main() {
    d := Dog{
        Animal: Animal{Name: "Rex", Age: 3},
        Breed:  "German Shepherd",
    }

    fmt.Println(d.Name)       // ✅ продвижение поля из Animal
    fmt.Println(d.Breathe())  // ✅ продвижение метода из Animal
    fmt.Println(d.Describe()) // ✅ метод Dog (не Animal)
    fmt.Println(d.Animal.Describe()) // ✅ явный вызов Animal.Describe
}
```

**Продвижение методов (method promotion):** методы встроенного типа становятся методами внешнего типа — но это не полиморфизм! `Dog` не является `Animal` в смысле интерфейса, если `Animal` — конкретный тип.

---

### Embedding + Interface = настоящий полиморфизм

```go
type Describer interface {
    Describe() string
}

type Animal struct{ Name string }
func (a Animal) Describe() string { return a.Name }

type Dog struct {
    Animal
    Breed string
}
// Dog.Describe() продвинут из Animal → Dog реализует Describer автоматически

type Cat struct {
    Animal
    Indoor bool
}
// Cat.Describe() продвинут из Animal → Cat тоже реализует Describer

func PrintAll(items []Describer) {
    for _, item := range items {
        fmt.Println(item.Describe())
    }
}

func main() {
    items := []Describer{
        Dog{Animal: Animal{"Rex"}, Breed: "Husky"},
        Cat{Animal: Animal{"Whiskers"}, Indoor: true},
    }
    PrintAll(items)
}
```

---

### Множественное "наследование" через несколько embedding

```go
type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println("[LOG]", msg) }

type Validator struct{}
func (v Validator) Validate(s string) bool { return len(s) > 0 }

type Service struct {
    Logger    // методы Log доступны напрямую
    Validator // методы Validate доступны напрямую
    name string
}

func (s Service) Process(input string) {
    if !s.Validate(input) { // из Validator
        s.Log("invalid input") // из Logger
        return
    }
    s.Log("processing: " + input)
}
```

---

### Ловушка: embedding не даёт полиморфизм по базовому типу

```go
type Base struct{}
func (b Base) Hello() { fmt.Println("base") }

type Child struct{ Base }

// Base не является интерфейсом — Child нельзя использовать там, где ожидается Base
func process(b Base) { b.Hello() }

func main() {
    c := Child{}
    // process(c) // ❌ compile error: cannot use Child as Base
    process(c.Base) // ✅ только так
}
```

Это принципиальное отличие от наследования в Java/C++.

---

## Частые вопросы на интервью

**Q: Поддерживает ли Go ООП?**
> Go поддерживает инкапсуляцию (unexported поля), полиморфизм и абстракцию (интерфейсы), но не наследование. Вместо наследования — композиция через embedding. Философия: "предпочитай композицию наследованию".

**Q: Чем Go отличается от Java в плане ООП?**
> 1. Нет классов — есть структуры + методы на любом типе
> 2. Нет явного `implements` — интерфейсы реализуются неявно (утиная типизация)
> 3. Нет наследования — только embedding (и это не то же самое)
> 4. Нет `abstract class` — только интерфейсы
> 5. Видимость на уровне пакета, не класса

**Q: В чём разница между embedding и наследованием?**
> - Наследование: `Child` является `Parent`, можно использовать везде, где ожидается `Parent`
> - Embedding: Child **содержит** Base, методы Base **продвигаются**, но Child ≠ Base с точки зрения типов
> - Embedding — это делегирование, а не субтипизация

**Q: Как реализовать abstract class в Go?**
> Никак напрямую. Используют комбинацию: интерфейс (обязательные методы) + базовую структуру с общей логикой, которую встраивают.

```go
type AbstractProcessor interface {
    Process(data string) error // обязательно реализовать
}

type BaseProcessor struct{} // общая логика
func (b BaseProcessor) Log(msg string) { fmt.Println(msg) }
func (b BaseProcessor) Validate(s string) bool { return len(s) > 0 }

// "Конкретный класс" — встраивает базу и реализует интерфейс
type EmailProcessor struct {
    BaseProcessor
}
func (e EmailProcessor) Process(data string) error {
    e.Log("processing email: " + data) // из BaseProcessor
    return nil
}
```

**Q: Что такое утиная типизация (duck typing)?**
> "Если оно ходит как утка и крякает как утка — это утка." В Go: если тип имеет все методы интерфейса — он его реализует, без явного объявления. Это структурная типизация (в отличие от номинальной в Java).
