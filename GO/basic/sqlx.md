sqlx = database/sql + 结构体自动映射 + 命名参数 + 批量操作
## 结构体自动映射
```go
import "github.com/jmoiron/sqlx"

type User struct {
    ID   int    `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

// 自动映射到结构体！
var user User
err := db.Get(&user, "SELECT * FROM users WHERE id = ?", 1)

// 查询多条记录
var users []User
err = db.Select(&users, "SELECT * FROM users WHERE age > ?", 18)
```

## 参数命名
```go
// 标准库不支持命名参数！
_, err := db.NamedExec(
    "INSERT INTO users (name, age) VALUES (:name, :age)",
    map[string]interface{}{
        "name": "Alice",
        "age":  25,
    },
)
```

## Context支持
```go
var user User
err := db.GetContext(ctx, &user, "SELECT * FROM users WHERE id = ?", 1)

// 其他 Context 方法：
db.SelectContext(ctx, &users, query, args...)
db.NamedExecContext(ctx, query, arg)
db.RebindContext(ctx, query) // 用于不同数据库方言
```

## 与标准库`sql`的对比
| 特性 | `database/sql` | `sqlx` |
| :--- | :--- | :--- |
| 维护方 | Go 官方 | 社区 (jmoiron) |
| 结构体映射 | ❌ 需要手动 Scan | ✅ 自动映射 (`Get`, `Select`) |
| 命名参数 | ❌ 不支持 | ✅ 支持 (`NamedExec`, `NamedQuery`) |
| Context 支持 | ✅ 原生支持 | ✅ 完全支持 |
| 学习成本 | 低（但代码繁琐） | 中（需要学标签语法） |
| 性能 | 最高（零额外开销） | 略低（反射开销，但可忽略） |
| 适用场景 | 简单查询、极致性能 | 大多数业务场景 |

