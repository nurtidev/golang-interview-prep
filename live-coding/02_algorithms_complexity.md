# Алгоритмы: оценка сложности и основные паттерны

## Как оценивать сложность (Big O)

### Что такое Big O

Big O описывает **верхнюю границу** роста времени выполнения или объёма памяти при увеличении входных данных `n`.

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

### Правила подсчёта

**1. Константы не важны**
```go
// O(3n) → O(n)
for i := 0; i < n; i++ { ... }
for i := 0; i < n; i++ { ... }
for i := 0; i < n; i++ { ... }
```

**2. Оставляем только доминирующий член**
```go
// O(n² + n) → O(n²)
for i := 0; i < n; i++ {
    for j := 0; j < n; j++ { ... }  // n²
}
for i := 0; i < n; i++ { ... }      // n — незначимо
```

**3. Вложенные циклы — перемножаем**
```go
// O(n * m)
for i := 0; i < n; i++ {
    for j := 0; j < m; j++ { ... }
}
```

**4. Последовательные блоки — складываем (потом берём макс)**
```go
// O(n) + O(n log n) → O(n log n)
sort.Slice(arr, ...)              // O(n log n)
for _, v := range arr { ... }    // O(n)
```

**5. Рекурсия — считаем через дерево вызовов**
```go
// T(n) = 2T(n/2) + O(n)  → O(n log n)  — merge sort
// T(n) = T(n-1) + O(1)   → O(n)         — linear recursion
// T(n) = 2T(n-1) + O(1)  → O(2ⁿ)        — fibonacci без мемоизации
```

### Мастер-теорема (для рекурсий вида T(n) = aT(n/b) + O(n^d))

| Условие     | Сложность      | Пример          |
|-------------|----------------|-----------------|
| d > log_b a | O(n^d)         | —               |
| d = log_b a | O(n^d · log n) | merge sort      |
| d < log_b a | O(n^log_b a)   | binary tree DFS |

### Пространственная сложность (Space)

```go
// O(n) — создаём новый массив
result := make([]int, n)

// O(1) — только переменные
sum := 0
for _, v := range arr { sum += v }

// O(n) — рекурсия, стек вызовов глубиной n
func factorial(n int) int {
    if n <= 1 { return 1 }
    return n * factorial(n-1)
}

// O(log n) — рекурсия binary search, глубина log n
```

---

## Основные алгоритмические паттерны

---

### 1. Two Pointers (два указателя)

**Когда применять:** массив/строка отсортированы или нужно найти пару/тройку элементов.

**Сложность:** O(n) вместо O(n²)

```go
// Паттерн 1: с двух концов → сходятся к центру
left, right := 0, len(arr)-1
for left < right {
    if условие {
        left++
    } else {
        right--
    }
}

// Паттерн 2: оба идут слева → разной скоростью (slow/fast)
slow, fast := 0, 0
for fast < len(arr) {
    if условие {
        arr[slow] = arr[fast]
        slow++
    }
    fast++
}
```

**Задачи:**
- Two Sum в отсортированном массиве
- Удалить дубликаты из отсортированного массива
- Container With Most Water
- 3Sum

---

### 2. Sliding Window (скользящее окно)

**Когда применять:** подмассив/подстрока с условием на непрерывный диапазон.

**Сложность:** O(n)

```go
// Фиксированный размер окна k
sum := 0
for i := 0; i < k; i++ { sum += arr[i] }
maxSum := sum
for i := k; i < len(arr); i++ {
    sum += arr[i] - arr[i-k]  // добавляем правый, убираем левый
    maxSum = max(maxSum, sum)
}

// Динамический размер окна (expand/shrink)
left := 0
window := make(map[byte]int)
for right := 0; right < len(s); right++ {
    window[s[right]]++                    // расширяем
    for условие_нарушено {
        window[s[left]]--                 // сжимаем
        left++
    }
    result = max(result, right-left+1)
}
```

**Задачи:**
- Longest Substring Without Repeating Characters
- Minimum Window Substring
- Maximum Sum Subarray of Size K
- Permutation in String

---

### 3. Binary Search (бинарный поиск)

**Когда применять:** отсортированный массив, поиск границы, "минимальный X при котором условие выполняется".

**Сложность:** O(log n)

```go
// Классический: найти элемент
lo, hi := 0, len(arr)-1
for lo <= hi {
    mid := lo + (hi-lo)/2  // избегаем overflow
    if arr[mid] == target {
        return mid
    } else if arr[mid] < target {
        lo = mid + 1
    } else {
        hi = mid - 1
    }
}

// Поиск левой границы (первый элемент >= target)
lo, hi := 0, len(arr)
for lo < hi {
    mid := lo + (hi-lo)/2
    if arr[mid] < target {
        lo = mid + 1
    } else {
        hi = mid
    }
}
// lo — первый индекс >= target

// Бинарный поиск по ответу: "найди минимальный k при котором f(k) = true"
lo, hi := minK, maxK
for lo < hi {
    mid := lo + (hi-lo)/2
    if feasible(mid) {
        hi = mid
    } else {
        lo = mid + 1
    }
}
```

**Задачи:**
- Search in Rotated Sorted Array
- Find First and Last Position
- Koko Eating Bananas (поиск по ответу)
- Capacity To Ship Packages

---

### 4. BFS (обход в ширину)

**Когда применять:** кратчайший путь в невзвешенном графе/дереве, обход по уровням.

**Сложность:** O(V + E)

```go
type Point struct{ x, y int }

func bfs(grid [][]int, start Point) int {
    queue := []Point{start}
    visited := map[Point]bool{start: true}
    steps := 0

    for len(queue) > 0 {
        size := len(queue)          // обрабатываем уровень целиком
        for i := 0; i < size; i++ {
            curr := queue[0]
            queue = queue[1:]

            if достигли_цели(curr) { return steps }

            for _, dir := range dirs {
                next := Point{curr.x + dir[0], curr.y + dir[1]}
                if valid(next) && !visited[next] {
                    visited[next] = true
                    queue = append(queue, next)
                }
            }
        }
        steps++
    }
    return -1
}
```

**Задачи:**
- Word Ladder
- Shortest Path in Binary Matrix
- Rotting Oranges
- Minimum Depth of Binary Tree

---

### 5. DFS (обход в глубину)

**Когда применять:** перебор всех путей, топологическая сортировка, связные компоненты, backtracking.

**Сложность:** O(V + E)

```go
// Итеративный DFS
func dfs(graph [][]int, start int) {
    stack := []int{start}
    visited := map[int]bool{start: true}
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        for _, next := range graph[node] {
            if !visited[next] {
                visited[next] = true
                stack = append(stack, next)
            }
        }
    }
}

// Рекурсивный DFS с backtracking
func dfs(node int, path []int, visited []bool) {
    if достигли_цели(node) {
        results = append(results, append([]int{}, path...))
        return
    }
    for _, next := range graph[node] {
        if !visited[next] {
            visited[next] = true
            path = append(path, next)
            dfs(next, path, visited)
            path = path[:len(path)-1]  // BACKTRACK
            visited[next] = false
        }
    }
}
```

**Задачи:**
- Number of Islands
- Clone Graph
- Course Schedule (cycle detection)
- All Paths From Source to Target

---

### 6. Backtracking

**Когда применять:** генерация всех комбинаций/перестановок, поиск с откатом.

**Сложность:** обычно O(2ⁿ) или O(n!)

```go
// Шаблон backtracking
var result [][]int

func backtrack(start int, current []int, candidates []int) {
    if условие_завершения {
        result = append(result, append([]int{}, current...))  // копируем!
        return
    }
    for i := start; i < len(candidates); i++ {
        if !подходит(candidates[i]) { continue }

        current = append(current, candidates[i])    // выбираем
        backtrack(i+1, current, candidates)         // рекурсия
        current = current[:len(current)-1]          // откатываем
    }
}

// Типичные вариации:
// - Subsets: start=i+1, не пропускаем элементы
// - Combinations: start=i+1, с ограничением по сумме
// - Permutations: start=0, used[i] флаг
// - Combination Sum с повторами: start=i (не i+1)
```

**Задачи:**
- Subsets, Subsets II
- Permutations
- Combination Sum
- N-Queens, Sudoku Solver

---

### 7. Dynamic Programming (динамическое программирование)

**Когда применять:** задача разбивается на перекрывающиеся подзадачи, есть оптимальная подструктура.

**Сложность:** зависит от размерности dp-таблицы

```go
// Признаки DP-задачи:
// - "минимальное/максимальное количество..."
// - "сколькими способами можно..."
// - "возможно ли достичь..."

// Шаблон 1D DP
dp := make([]int, n+1)
dp[0] = базовый_случай
for i := 1; i <= n; i++ {
    dp[i] = f(dp[i-1], dp[i-2], ...)
}

// Шаблон 2D DP (например, LCS)
dp := make([][]int, m+1)
for i := range dp { dp[i] = make([]int, n+1) }
for i := 1; i <= m; i++ {
    for j := 1; j <= n; j++ {
        if s1[i-1] == s2[j-1] {
            dp[i][j] = dp[i-1][j-1] + 1
        } else {
            dp[i][j] = max(dp[i-1][j], dp[i][j-1])
        }
    }
}

// Оптимизация памяти: если dp[i] зависит только от dp[i-1] → O(n) space
prev, curr := 0, 0
for i := 1; i <= n; i++ {
    curr = f(prev)
    prev = curr
}
```

**Классические DP-задачи:**

| Задача               | Формула перехода                              | Сложность     |
|----------------------|-----------------------------------------------|---------------|
| Fibonacci            | `dp[i] = dp[i-1] + dp[i-2]`                  | O(n) / O(1)   |
| Climbing Stairs      | `dp[i] = dp[i-1] + dp[i-2]`                  | O(n) / O(1)   |
| Coin Change          | `dp[i] = min(dp[i-coin]+1)`                   | O(n·k)        |
| 0/1 Knapsack         | `dp[i][w] = max(dp[i-1][w], dp[i-1][w-wi]+vi)` | O(n·W)     |
| LCS                  | `dp[i][j] = dp[i-1][j-1]+1 или max(...)`      | O(m·n)        |
| Edit Distance        | `dp[i][j] = min(insert, delete, replace)`     | O(m·n)        |
| Longest Inc. Subseq  | `dp[i] = max(dp[j]+1) где arr[j]<arr[i]`      | O(n²) / O(n log n) |

---

### 8. Heap / Priority Queue

**Когда применять:** K-й наибольший/наименьший, top-K элементов, merge K sorted lists.

**Сложность:** O(n log k) для top-K задач

```go
import "container/heap"

// Min-heap для top-K наибольших элементов
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
    old := *h; n := len(old)
    x := old[n-1]; *h = old[:n-1]; return x
}

func topKFrequent(nums []int, k int) []int {
    h := &MinHeap{}
    heap.Init(h)
    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)  // убираем минимум, оставляем k наибольших
        }
    }
    return *h
}
```

**Задачи:**
- Kth Largest Element
- Top K Frequent Elements
- Merge K Sorted Lists
- Find Median from Data Stream

---

### 9. Prefix Sum (префиксные суммы)

**Когда применять:** множественные запросы суммы диапазона, подмассив с нужной суммой.

**Сложность:** O(n) построение, O(1) запрос

```go
// Построение
prefix := make([]int, len(arr)+1)
for i, v := range arr {
    prefix[i+1] = prefix[i] + v
}

// Запрос суммы на [l, r] включительно
sum := prefix[r+1] - prefix[l]

// Подмассив с суммой равной target
// prefix[j] - prefix[i] = target  →  prefix[i] = prefix[j] - target
seen := map[int]int{0: 1}
curr, count := 0, 0
for _, v := range arr {
    curr += v
    count += seen[curr-target]
    seen[curr]++
}
```

**Задачи:**
- Range Sum Query
- Subarray Sum Equals K
- Continuous Subarray Sum
- Product of Array Except Self

---

### 10. Monotonic Stack (монотонный стек)

**Когда применять:** "следующий больший/меньший элемент", площадь прямоугольника, температура.

**Сложность:** O(n)

```go
// Next Greater Element
result := make([]int, len(arr))
stack := []int{}  // индексы

for i := 0; i < len(arr); i++ {
    // пока текущий элемент больше верхушки стека
    for len(stack) > 0 && arr[i] > arr[stack[len(stack)-1]] {
        idx := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result[idx] = arr[i]  // arr[i] — следующий больший для idx
    }
    stack = append(stack, i)
}
// оставшиеся в стеке → нет большего элемента → result[idx] = -1
```

**Задачи:**
- Daily Temperatures
- Next Greater Element
- Largest Rectangle in Histogram
- Trapping Rain Water

---

## Быстрые подсказки: какой паттерн выбрать

| Признак задачи | Паттерн |
|---|---|
| Отсортированный массив, пара элементов | Two Pointers |
| Непрерывная подстрока/подмассив | Sliding Window |
| "Найди в отсортированном" / "минимальный X при котором..." | Binary Search |
| Кратчайший путь в графе/сетке | BFS |
| Все пути / связные компоненты | DFS |
| Все комбинации/перестановки | Backtracking |
| Мин/макс, количество способов | DP |
| Top-K, K-й наибольший | Heap |
| Сумма диапазонов, подмассив с суммой K | Prefix Sum |
| Следующий больший/меньший элемент | Monotonic Stack |

---

## Типичные ловушки на интервью

### Integer overflow
```go
mid := lo + (hi-lo)/2  // правильно
mid := (lo + hi) / 2   // переполнение если lo+hi > MaxInt32
```

### Копирование слайса
```go
// Backtracking: ВСЕГДА копируем перед добавлением в result
result = append(result, append([]int{}, current...))
// или
tmp := make([]int, len(current))
copy(tmp, current)
result = append(result, tmp)
```

### Off-by-one в бинарном поиске
```go
// Если ищем левую границу: hi = len(arr), условие lo < hi
// Если ищем точное совпадение: hi = len(arr)-1, условие lo <= hi
```

### nil map
```go
var m map[string]int  // nil, запись вызовет panic
m = make(map[string]int)  // правильно
```

### Цикл в графе
```go
// Всегда помечаем посещённые узлы ДО добавления в очередь/стек
visited[next] = true
queue = append(queue, next)
// а не после pop — иначе добавим один узел несколько раз
```

---

## Оценка своего решения на интервью

Скажи вслух:
1. **Brute force:** "Наивное решение — O(n²), просто перебираем все пары"
2. **Оптимизация:** "Можно улучшить до O(n) с помощью хэш-таблицы / O(n log n) с сортировкой"
3. **Trade-offs:** "Тратим O(n) памяти, зато выигрываем по времени"
4. **Edge cases:** пустой массив, один элемент, все одинаковые, отрицательные числа

```
Время:  O(?)
Память: O(?)
Edge cases: [], [1], все одинаковые, INT_MIN/INT_MAX
```
