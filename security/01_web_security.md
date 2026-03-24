# Web-безопасность

---

## 1. OWASP Top 10 — краткий обзор

| # | Уязвимость | Пример |
|---|-----------|--------|
| A01 | Broken Access Control | Пользователь A видит данные пользователя B |
| A02 | Cryptographic Failures | Пароли в MD5, HTTP вместо HTTPS |
| A03 | **Injection** (SQL, Command) | `SELECT * WHERE id = '1; DROP TABLE users'` |
| A04 | Insecure Design | Отсутствие rate limiting на login |
| A05 | Security Misconfiguration | Debug mode в prod, дефолтные пароли |
| A06 | Vulnerable Components | Зависимость с известной CVE |
| A07 | Auth Failures | Слабые пароли, нет MFA, session fixation |
| A08 | Data Integrity Failures | Deserializing untrusted data |
| A09 | Logging Failures | Не логируем попытки взлома |
| A10 | SSRF | Сервер делает запрос на внутренний адрес по user input |

---

## 2. SQL Injection

### Как работает

```sql
-- Уязвимый запрос (конкатенация строк):
query := "SELECT * FROM users WHERE email = '" + email + "'"

-- Если email = "' OR '1'='1":
-- SELECT * FROM users WHERE email = '' OR '1'='1'
-- Вернёт ВСЕХ пользователей!

-- Если email = "'; DROP TABLE users; --":
-- SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
-- Удалит таблицу!
```

### Защита: параметризованные запросы

```go
// BAD: конкатенация
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)
rows, err := db.Query(query)

// GOOD: параметры (placeholder)
rows, err := db.QueryContext(ctx,
    "SELECT id, name, email FROM users WHERE email = $1 AND active = $2",
    email, true,
)

// GOOD: sqlx именованные параметры
type Filter struct {
    Email  string `db:"email"`
    Active bool   `db:"active"`
}
rows, err := db.NamedQueryContext(ctx,
    "SELECT * FROM users WHERE email = :email AND active = :active",
    Filter{Email: email, Active: true},
)

// GOOD: ORM (gorm, ent) автоматически параметризует
db.Where("email = ? AND active = ?", email, true).Find(&users)
```

### Second-order SQL Injection

```go
// Данные сохранены в БД "чисто", но потом используются в запросе небезопасно:
username := getUserFromDB(id)  // "admin'--"
query := "SELECT * FROM logs WHERE user = '" + username + "'"
// Уязвимо! Даже если данные пришли из БД — параметризуй всегда
```

---

## 3. XSS (Cross-Site Scripting)

### Типы

```
Stored XSS:  Зловредный скрипт сохранён в БД → выдаётся всем пользователям
Reflected XSS: Скрипт в URL параметре → выдаётся только жертве (нужна ссылка)
DOM XSS:     JavaScript на странице записывает user input в DOM без санитизации
```

### Защита в Go

```go
import "html/template"

// BAD: text/template — не экранирует HTML
import "text/template"
tmpl.Execute(w, userInput)  // <script>alert(1)</script> выполнится!

// GOOD: html/template — автоматически экранирует
import "html/template"
// <script> → &lt;script&gt; — браузер покажет текст, не выполнит скрипт
tmpl.Execute(w, userInput)

// Для JSON API: Content-Type: application/json
// Браузер не будет интерпретировать ответ как HTML

// CSP (Content Security Policy) заголовок — дополнительная защита
w.Header().Set("Content-Security-Policy",
    "default-src 'self'; script-src 'self'; object-src 'none'")

// Для хранения пользовательского HTML (rich text editor):
// Используй bleach/bluemonday для whitelist санитизации
import "github.com/microcosm-cc/bluemonday"
p := bluemonday.UGCPolicy()  // User-Generated Content policy
safe := p.Sanitize(userHTML)  // удаляет <script>, опасные атрибуты
```

---

## 4. JWT (JSON Web Tokens)

### Структура

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header (base64)
.eyJ1c2VySWQiOjEsInJvbGUiOiJ1c2VyIn0   ← Payload (base64)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV    ← Signature (HMAC/RSA)

Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"userId": 1, "role": "user", "exp": 1735689600, "iat": 1735603200}
```

**Важно:** Payload — это base64, НЕ шифрование. Любой может декодировать и прочитать. Не клади пароли, секреты.

### Реализация в Go

```go
import "github.com/golang-jwt/jwt/v5"

var secretKey = []byte(os.Getenv("JWT_SECRET"))

type Claims struct {
    UserID int64  `json:"userId"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// Создание токена
func GenerateToken(userID int64, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

// Валидация токена
func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            // КРИТИЧНО: проверяем алгоритм!
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return secretKey, nil
        })

    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token claims")
    }
    return claims, nil
}

// Middleware
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if !strings.HasPrefix(authHeader, "Bearer ") {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        tokenString := strings.TrimPrefix(authHeader, "Bearer ")
        claims, err := ValidateToken(tokenString)
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        ctx := context.WithValue(r.Context(), ctxKeyUserID, claims.UserID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### JWT уязвимости

```
1. alg=none атака:
   Злоумышленник меняет alg на "none" → подпись не проверяется
   ЗАЩИТА: явно проверяй алгоритм (показано выше)

2. Слабый секрет:
   secretKey = "secret" → брутфорс за секунды
   ЗАЩИТА: минимум 256 бит случайных данных
   openssl rand -base64 32

3. Нет проверки exp:
   Токен "живёт вечно" если не проверять ExpiresAt
   ЗАЩИТА: jwt.ParseWithClaims проверяет автоматически

4. Хранение в localStorage:
   XSS может украсть токен
   ЗАЩИТА: httpOnly cookie (недоступна JS)
```

---

## 5. Аутентификация и пароли

### Хеширование паролей

```go
import "golang.org/x/crypto/bcrypt"

// Хешировать при регистрации (cost=12 — ~300ms, подбирай под нагрузку)
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(bytes), err
}

// Проверять при логине
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// НЕ ИСПОЛЬЗОВАТЬ для паролей:
// MD5, SHA1, SHA256 — слишком быстрые, брутфорс за часы/дни
// bcrypt, scrypt, argon2id — специально медленные (memory+CPU hard)

// argon2id — современная рекомендация (OWASP 2023):
import "golang.org/x/crypto/argon2"
hash := argon2.IDKey([]byte(password), salt, 1, 64*1024, 4, 32)
// time=1, memory=64MB, threads=4, keyLen=32
```

### Безопасное сравнение строк

```go
import "crypto/subtle"

// BAD: timing attack — по времени ответа можно угадать длину/содержание
if token == expectedToken { ... }

// GOOD: constant time comparison
if subtle.ConstantTimeCompare([]byte(token), []byte(expectedToken)) == 1 { ... }
```

---

## 6. Rate Limiting

```go
import "golang.org/x/time/rate"

// Token bucket rate limiter
limiter := rate.NewLimiter(rate.Limit(100), 200)  // 100 req/s, burst 200

func RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            w.Header().Set("Retry-After", "1")
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Per-IP rate limiting с sync.Map
type IPLimiter struct {
    limiters sync.Map
    rate     rate.Limit
    burst    int
}

func (l *IPLimiter) GetLimiter(ip string) *rate.Limiter {
    v, _ := l.limiters.LoadOrStore(ip,
        rate.NewLimiter(l.rate, l.burst))
    return v.(*rate.Limiter)
}

func (l *IPLimiter) Allow(ip string) bool {
    return l.GetLimiter(ip).Allow()
}

// Для распределённого rate limiting — Redis + sliding window:
// lua script для атомарного INCR + EXPIRE
```

---

## 7. SSRF (Server-Side Request Forgery)

```go
// Уязвимость: сервер делает HTTP запрос по URL из user input
// Атака: передать внутренний адрес (http://169.254.169.254/ — AWS metadata)

// BAD:
func fetchURL(w http.ResponseWriter, r *http.Request) {
    url := r.URL.Query().Get("url")
    resp, err := http.Get(url)  // опасно!
    // ...
}

// GOOD: whitelist разрешённых хостов
var allowedHosts = map[string]bool{
    "api.example.com": true,
    "cdn.example.com": true,
}

func validateURL(rawURL string) error {
    u, err := url.Parse(rawURL)
    if err != nil {
        return err
    }
    if u.Scheme != "https" {
        return errors.New("only https allowed")
    }
    hostname := u.Hostname()
    if !allowedHosts[hostname] {
        return fmt.Errorf("host %s not allowed", hostname)
    }
    return nil
}

// GOOD: кастомный transport с проверкой resolved IP
func safeHTTPClient() *http.Client {
    dialer := &net.Dialer{}
    transport := &http.Transport{
        DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
            conn, err := dialer.DialContext(ctx, network, addr)
            if err != nil {
                return nil, err
            }
            // Проверяем что resolved IP не приватный
            host, _, _ := net.SplitHostPort(addr)
            ip := net.ParseIP(host)
            if isPrivateIP(ip) {
                conn.Close()
                return nil, errors.New("private IPs not allowed")
            }
            return conn, nil
        },
    }
    return &http.Client{Transport: transport}
}

func isPrivateIP(ip net.IP) bool {
    privateRanges := []string{
        "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
        "127.0.0.0/8", "169.254.0.0/16",  // link-local (AWS metadata!)
        "::1/128", "fc00::/7",
    }
    for _, cidr := range privateRanges {
        _, network, _ := net.ParseCIDR(cidr)
        if network.Contains(ip) {
            return true
        }
    }
    return false
}
```

---

## 8. Безопасные HTTP заголовки

```go
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        h := w.Header()

        // Запрещает браузеру угадывать Content-Type
        h.Set("X-Content-Type-Options", "nosniff")

        // Защита от clickjacking
        h.Set("X-Frame-Options", "DENY")

        // Принудительный HTTPS на 1 год
        h.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

        // Content Security Policy
        h.Set("Content-Security-Policy",
            "default-src 'self'; img-src 'self' data:; script-src 'self'")

        // Не передавать Referer на другие домены
        h.Set("Referrer-Policy", "strict-origin-when-cross-origin")

        // Запрет доступа к мощным API браузера
        h.Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")

        next.ServeHTTP(w, r)
    })
}
```

---

## 9. CORS (Cross-Origin Resource Sharing)

```go
// Браузер блокирует запросы с другого origin по умолчанию
// CORS заголовки разрешают конкретные origins

func CORSMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    allowed := make(map[string]bool)
    for _, o := range allowedOrigins {
        allowed[o] = true
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")

            if allowed[origin] {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Credentials", "true")
                w.Header().Set("Vary", "Origin")  // важно для кешей!
            }

            if r.Method == http.MethodOptions {
                // Preflight request
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
                w.Header().Set("Access-Control-Max-Age", "86400")  // кешировать preflight
                w.WriteHeader(http.StatusNoContent)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// BAD: Access-Control-Allow-Origin: * с credentials — браузер заблокирует
// BAD: Access-Control-Allow-Origin: * — открывает API для любого сайта (не для APIs с auth)
```

---

## 10. Secrets Management

```go
// BAD: секреты в коде
const dbPassword = "supersecret"

// BAD: секреты в env файлах в репозитории
// .env → commit → GitHub → утечка

// GOOD: env variables через CI/CD
dbPassword := os.Getenv("DB_PASSWORD")
if dbPassword == "" {
    log.Fatal("DB_PASSWORD not set")
}

// GOOD: Vault / AWS Secrets Manager / GCP Secret Manager
import "github.com/hashicorp/vault/api"

func getSecret(path, key string) (string, error) {
    client, _ := api.NewClient(api.DefaultConfig())
    secret, err := client.Logical().Read(path)
    if err != nil {
        return "", err
    }
    return secret.Data[key].(string), nil
}

// GOOD: Kubernetes Secrets + mounted files
// secret монтируется как /var/secrets/db-password
password, _ := os.ReadFile("/var/secrets/db-password")

// Ротация секретов: не хардкодь TTL, поддерживай горячую перезагрузку
```

---

## 11. Input Validation

```go
// Валидировать на входе — не доверяй ничему от клиента
type CreateUserRequest struct {
    Name  string `json:"name"  validate:"required,min=2,max=100,alphanum"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age"   validate:"min=0,max=150"`
    Role  string `json:"role"  validate:"oneof=user admin moderator"`
}

import "github.com/go-playground/validator/v10"

var validate = validator.New()

func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", http.StatusBadRequest)
        return
    }

    if err := validate.Struct(req); err != nil {
        // Не возвращать внутренние детали — только что неверно
        http.Error(w, "validation failed: "+err.Error(), http.StatusBadRequest)
        return
    }
    // ...
}

// Path traversal protection:
func safeFilePath(base, userInput string) (string, error) {
    // filepath.Join нормализует ".." и т.д.
    path := filepath.Join(base, filepath.Clean("/"+userInput))
    if !strings.HasPrefix(path, base) {
        return "", errors.New("path traversal detected")
    }
    return path, nil
}
```

---

## Вопросы на собеседовании

### Q: Как защититься от SQL injection?

**Ответ:** Всегда параметризованные запросы — никогда конкатенация строк. В Go: `db.QueryContext(ctx, "SELECT ... WHERE id = $1", id)`. Библиотеки (pgx, sqlx, gorm) параметризуют автоматически при правильном использовании. Дополнительно: минимальные права БД (SELECT/INSERT только нужные таблицы), WAF как дополнительный слой. Нельзя полагаться только на эскейпинг — параметризация это единственно надёжный способ.

---

### Q: В чём разница между аутентификацией и авторизацией?

**Ответ:** Аутентификация — кто ты? (логин + пароль, JWT токен, mTLS сертификат). Авторизация — что тебе можно? (RBAC роли, ACL списки, политики). Типичная ошибка — проверять только аутентификацию и забывать авторизацию: "пользователь залогинен" ≠ "пользователь имеет право читать этот ресурс". В Go: middleware для auth (проверка JWT), отдельный слой для авторизации (casbin, OPA, или ручная проверка ролей).

---

### Q: Что такое CSRF? Как защититься?

**Ответ:** CSRF (Cross-Site Request Forgery) — злоумышленник заставляет браузер жертвы сделать запрос к сервису где она аутентифицирована. Пример: кнопка на evil.com делает POST /transfer на bank.com — браузер автоматически добавляет cookies bank.com. Защита: 1) CSRF токен — случайный токен в форме, сервер проверяет что он совпадает с session (не может подделать злоумышленник, т.к. SOP блокирует чтение ответов с другого origin); 2) SameSite=Strict/Lax на cookies — браузер не отправит cookie с cross-site запроса; 3) Проверка Origin/Referer заголовка. Для JSON API + Bearer токен — CSRF не актуален (браузер не добавляет Bearer автоматически).
