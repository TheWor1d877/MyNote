关键结论：不是完整复制整张表，而是在一条记录后面“挂一个历史版本链”


假设有一行数据：id=1, name='张三', balance=100
修改：将100变成90
数据库不会覆盖旧数据,而是：
   把旧数据 (100) 写入 Undo Log
   当前数据行改为 90，并增加两个隐藏字段
```text
当前数据行（最新版本）：
id=1, name=‘张三’, balance=90, DB_TRX_ID=事务A, DB_ROLL_PTR → 指向旧版本

                ↓ (通过指针)

Undo Log 中的旧版本：
id=1, name=‘张三’, balance=100, DB_TRX_ID=事务0, DB_ROLL_PTR → NULL
```

<span style="color:rgb(221, 85, 85)"> 所谓的“多个版本” = 当前行 + 一条通过指针串起来的 Undo Log 链表</span> 

## Undo Log作用
Undo Log = 用于“撤销”操作的日志，同时也充当历史版本仓库
它记录的是：如果你要回滚，应该把数据恢复成什么样子

作用	说明
事务回滚	如果你执行 ROLLBACK，数据库根据 Undo Log 把数据改回去
MVCC 多版本读	读旧数据时，沿着 DB_ROLL_PTR 去 Undo Log 里找历史版本