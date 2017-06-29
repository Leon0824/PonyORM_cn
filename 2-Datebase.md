# Datebase

在使用實體對象之前，你必須先創建一個 `Datebase` 對象。這個對象通過連接池管理著數據庫的連接。`Datebase` 對象是線程安全的，可以在你的應用的所有線程之間共享。通過`Datebase` 對象允許你直接是使用 SQL 操作數據庫，但是大多數時候都是通過操作實體對象讓 Pony 生成命令去修改數據庫的對應部分。Pony 允許你同時操作多個數據庫，但是一個實體對象只能對應一個數據庫。

將實體對象映射到數據庫可以分為四個步驟：

- 創建一個數據庫對象
- 創建跟這個數據庫關聯的實體對象
- 將數據庫對象綁定到指定的數據庫
- 映射實體對象到數據庫的表

## 創建數據庫對象
這一步我用最簡單的方法創建一個 `Database`類的實例：
`db = Database()`
雖然在這裏你就可以傳遞數據庫連接參數了，但是在下個階段使用 `db.bind()` 方法通常是更方便的。這樣你可以對測試環境和生產環境切換不同的數據庫。
`Database` 實例有一個 `Entity` 屬性作為實體類型聲明時的基類使用。

## 定義與數據庫對象關聯的實體類型
實體對象需要從 `Database` 對象的基類繼承。
```python
class MyEntity(db.Entity):
    attr1 = Required(str)
```
下一張我們將詳細討論實體對象定義的細節。現在我們先看下一步，映射實體對象到數據庫。

## 映射實體對象到數據庫
在我們映射實體對象到數據庫時，我們需要先連接到數據庫。這一步，我們需要使用 `bind()` 方法：
```python
db.bind('postgres', user='', password='', host='', database='')
```
第一個參數用來指定提供的數據庫。數據庫提供了一個模塊，封裝在 `pony.orm.dbproviders` 包中，這個模塊實現了對具體數據庫操作的能力。接下來的一個參數需要指定傳遞給一致性 DBAPI 中的 `connect()` 方法的參數。

## 數據庫支持模塊
目前，Pony 支持的操作系統有：SQLite，PostgreSQL，MySQL和 Oracle， Pony 提供相應的名字：'SQLite'，'PostgreSQL'，'MySQL'和 'Oracle'。可以很容易的擴展數據庫支持模塊。

在 `bind()` 執行期間，Pony 嘗試建立一個測試連接到數據庫。如果指定的參數不正確或者數據庫不可用，會拋出一個異常。當數據庫連接建立成功後，Pony 讀取數據庫的版本，然後歸還書庫據連接到連接池。

### SQLite

使用 SQLite 數據庫是使用 Pony 最簡單的方式，因為沒有必要去額外的建立一個數據庫系統 ——
 SQLite 數據庫系統已經包含在 Pyhton 的包中。初學者在交互環境中體驗 Pony 是最好的選擇。為了綁定一個數據庫對象到 SQLite 數據庫，你可以使用如下命令：
```python
db.bind('sqlite', 'filename', create_db=True)
```
這裏 'filename' 是需要存儲數據的 SQLite 數據庫的文件。文件名字可以是絕對路徑也可以是相對路徑。

>註
如果你指定了一個相對目錄，這個目錄將接在 Python 文件所在的文件夾下面（而不是當前工作目錄的下面）。我們這麽做是因為有些時候程序員沒有操作當前工作目錄的權限（例如在 mod_wsgi 應用中）。這個方法也可以讓程序員創建由不同獨立模塊組合起來的應用，每個模塊都可以有獨立的數據庫。

當在交互器中工作時， Pony 建議總是使用絕對路徑指定存儲文件。

如果參數 `create_db` 是 `True`，Pony 在指定文件不存在的時候將會嘗試創建文件。默認的 `create_db` 值是 `False`。

>註
一般情況下， SQLite 數據庫存儲在硬盤的文件上，但是也可以完全被存儲在內存中。如果是在交互器中玩玩 Pony，這種方式是非常方便的，但是你必須註意，所有的內存中的數據庫將在程序退出的時候丟失。而且內存中的數據庫在多線程的程序中也表現的不同，因為 SQLite 限制了所有的線程共享相同的連接來操作數據庫。

>通過用 `:memory:` 替代文件名，可以在內純中創建數據庫。
`db.bind('sqlite', ':memory:')`
 如果是在內存中創建數據庫，`create_db`也不要設置了。

 
>註
SQLite 默認不檢查外鍵限制。自 release 0.4.9 起，可以通過發送命令 `PRAGMA foreign_keys = ON;` 來讓 Pony 十種開啟外鍵檢測。

### PostgreSQL
Pony 使用 psycopg2 驅動 PostgreSQL 。綁定到 PostgreSQL 數據庫對象可以使用如下方法：
`db.bind('postgres', user='', password='', host='', database='')`
接下來的所有參數將會傳遞給 `psycopg2.connect()` 方法。通過 [psycopg2.connect documentation](http://initd.org/psycopg/docs/module.html#psycopg2.connect) 可以查到更多的可以傳遞的參數。

### MySQL
`db.bind('mysql', host='', user='', passwd='', db='')`
Pony 嘗試使用 MySQLdb 驅動 MySQL，如果這個模塊不能導入，Pony 會嘗試使用 pymysql。耕作內容參見：[MySQLdb](http://mysql-python.sourceforge.net/MySQLdb.html#functions-and-attributes) and [pymysql](https://pypi.python.org/pypi/PyMySQL)。

### Oracle
`db.bind('oracle', 'user/password@dsn')`
Pony 使用 cx_Oracle 驅動來連接 Oracle 數據庫。更多關於可以傳遞到數據庫連接的函數的參數請參考[這裏](http://cx-oracle.sourceforge.net/html/module.html)。

## 映射實體對象到數據庫表
`Database`對象創建了，實體對象定義了，數據庫也連接了，下一步就是映射實體對象到數據庫表。
`db.generate_mapping(check_tables=True, create_tables=False)`
如果 `create_tables` 被設置為 `True`，那麽 Pony 會嘗試創建不存在的表。`create_tables` 的默認值是 `False` 因為大多數情況下表都是存在的。Pony 會自動生成數據庫表和列的名字，但是如果你願意，可以重寫這個行為。具體細節參考 自定義映射 章節。這個參數也讓 Pony 檢查外鍵和索引是否存在，如果不存在則創建它們。

如果傳遞了 `create_tables` 參數，Pony 做了一個簡單的檢查：發送一個檢測實體對象的表和列名字均存在的 SQL 命令。但是，不檢測表是否有額外的列，或者列的類型和實體類型定義的不匹配。通過傳遞 `check_tables=True` 參數可以關閉檢測過程。當你想使用 `db.create_tables()` 方法在稍後的時候再生成映射和表的時候，這個設置是有用的。

##　更早的數據庫綁定
通過在創建數據庫對象的時候傳遞數據可參數，可以合並“創建數據庫對象”和“綁定數據庫對象到指定的數據庫”這兩步為一步。
```python
db = Database('sqlite', 'filename', create_db=True)

db = Database('postgres', user='', password='', host='', database='')

db = Database('mysql', host='', user='', passwd='', db='')

db = Database('oracle', 'user/password@dsn')
```
參數的設置都跟傳遞到 `bind()` 方法的相同。如果在數據庫對象創建啊時傳遞了這些參數，那麽之後就不用再調用 `bind()` 方法了 —— 數據庫對象和數據庫已經綁定了。

##　數據庫對象的方法和屬性
class Datebase

- generate_mapping(check_tables=True, create_tables=False)
映射聲明的屬性到數據庫中相應的表。`create_tables=True` - 創建不存在的數據表，外鍵和索引。`check_tables=False` 關閉表校驗。只檢測數據庫表名稱和屬性名稱與實體對象的聲明匹配，不檢測數據表擁有額外的列和列的屬性與聲明不匹配的情況。

- create_tables()
檢查實體的對象的映射和創建不存在的表，Pony 同時也檢查外鍵和索引是否存在。

- drop_all_tables(with_all_data=False)
刪除所有存在映射關系的中表。當這個方法以無參數形式調用的時候，Pony 只會在所有的表中都沒有數據的時候刪除這些表。為了防止誤刪，只要任何數據表中有數據，那麽這個方法會拋出一個 `TableIsNotEmpty` 異常，為了刪除帶數據的表，需要傳遞參數 `with_all_data=False`。

- drop_table(table_name, if_exists=False, with_all_data=False)
刪除 `table_name` 指定的表，如果表不存在，拋出 `TableDoesNotExist` 異常。註意，table_name 是大小寫敏感的。
可以傳遞實體對象的類名作為 `table_name`，這種情況下，Pony 將會嘗試刪除和實體對象關聯的表。
如果參數 `if_exists` 被設置為 `True`，即使表不存在也不會拋出 `TableDoesNotExist` 異常。當表非空的時候會拋出 `TableIsNotEmpty` 異常。
使用參數 `with_all_data=True` 可以刪除包含數據的表。如果需要刪除實體對象對應的表，可以調用實體對象的 `drop_tables()` 方法，它總會刪除正確對應的大小寫拼寫的名字的表。

### 傳輸相關的方法
class Database

- commit()
通過當前 `db_session` 的 `flush()` 方法，保存所有的變化並提交事務到數據庫。
程序員可以在一個 `db_session` 中多次調用 `commit()`。這種情況下，`db_session` 緩存了上次提交之後的對對象的所有操作。這允許 Pony 使用相同的一個對象實現鏈式的事務調用。當 `db_session` 調用結束或者或者事務回滾的時候，緩存會被清空。

- rollback()
回滾事務和清空 `db_session`中的緩存。

- flush()
更新 `db_session` 累積 的緩存到數據庫。你可能永遠不需要手動調用這個方法。Pony 在執行：select()，get()， exists()， execute() 和 commit()
等方法之前會自動的保存累積的。

### Database 對象屬性
- Enity
這個屬性提供了一個應該被所有需要映射到制定數據庫的實體對象的類型繼承的基類。
```python
db = Database()

class Person(db.Entity):
    name = Required(str)
    age = Required(int)
```
- last_sql
這是一個保留最後一條 SQL 命令的只讀屬性。它可以被用來調試程序。

### 使用原始數據的方法
class Database

- select(sql)
在數據庫中執行 SQL 語句，返回一個元組的列表。Pony 會從連接池中取出一個數據庫連接，執行完查詢語句後再將連接送回到數據庫中。SQL 語句中的 `select` 單詞是可以省略的 —— Pony 會在必要的時候幫你加上。
`select("* from Person")`
我們這麽做因為這樣方法的名字已經說明了語句的用途，這樣查詢語句看起來會比較簡明。如果一個查詢語句的返回結果是只有一列，那麽，為了方便，返回結果將會一個列表，沒有包含元組。
`db.select("name from Person")`
以上語句返回: [“John”, “Mary”, “Bob”]
如果一個查詢語句返回了多列，而且數據表的列的名字可以作為 Python identifiers，這時也可以通過類似調用屬性的方法調用。
```python
for row in db.select("name, age from Person"):
    print row.name, row.age
```
Pony 有一個對 `select` 方法能夠返回列的數量的限制。這個限制是通過 `pony.options.MAX_FETCH_COUNT` 參數指定的（默認是1000）。如果一個 `select` 返回超過 `MAX_FETCH_COUNT` Pony 拋出一個 `TooManyRowsFound`異常。你可以改變這個值，但是我們不建議這麽做，因為如果一個查詢語句返回超過 1000 列，那麽很可能是這個應用有設計上的問題。`select` 的返回結果是存儲在內存中的，如果列的數量是非常大的，應用可能面臨著性能問題。

- get(sql)
如果你只需要從數據庫中拿出一行數據或者一個值，可以使用 `get` 方法：
`age = db.get("age from Person where id = $id")`
SQL 語句中的 `select` 單詞可以被省略 —— Pony 將會在必要的時候自動添加 select 關鍵詞。如果 Person 數據表中存在指定的 id，age 變量將會賦予 int 類型的對應的值。`Get`方法假定返回一行的值，如果查詢結果為空，會拋出 `RowNotFound` 異常，如果查詢結果不只一行，會拋出 `MultipleRowsFound` 異常。

如果你需要選擇多個列，可以使用這個方法：
`name, age = db.get("name, age from Person where id = $id")`

如果你的的查詢返回許多列，你可以將結果存入到變量中，還是可以使用 `select`方法中描述的相同的方法來操作。

在執行 `get` 方法前，Pony 使用 `flush()` 方法刷新所有的緩存中的變化。

- exists(sql)
`exists` 方法是用來檢查指定參數在數據庫中是否存在至少一個數據，返回結果可能是 `True` 或者 `False`。
```python
if db.exists("* from Person where name = $name"):
    print "Person exists in the database"
```
SQL 命令的的 select 單詞可以省略。
在執行此方法前，Pony 使用 `flush()` 方法刷新所有的緩存中的變化。

- insert(table_name, returning=None, **kwargs)
數據庫中插入新的數據行。這個命令旁路身份映射緩存[^identity-map-cache]，可以用來在創建很多對象但是不需要同時操作它們的時候提升性能。你也可以使用 `db.execute()`達到相同的目的。如果我們需要在創建的同時右又要操作它們，那麽創建實體對象的實例，然後通過 Pony 用 `commit()` 命令保存它們將是更好的方法。

`table_name` —— 是要插入數據的數據表的名字，大小寫敏感的。

`returning` 參數讓你可以指定自動生成的主鍵列。例如你想要 `insert` 方法返回數據庫自動生成的數據，你應該指定主鍵列的名字。
`new_id = db.insert("Person", name="Ben", age=33, returning='id')`

- execute(sql)
這個方法讓你可以執行任意的（原生的）SQL語句。
```python
cursor = db.execute("""create table Person (
                           id integer primary key autoincrement,
                           name text,
                           age integer
                    )""")
name, age = "Ben", 33
cursor = db.execute("insert into Person (name, age) values ($name, $age)")
```
基於不同的 DBAPI，Pony 使用 `$` 符號統一的傳遞所有的參數。在上例中，我們就傳遞了 `name` 和 `age` 兩個參數到命令中。
命令中使用 Python 表達式也是可以的，例如：
```python
x = 10
a = 20
b = 30
db.execute("SELECT * FROM Table1 WHERE column1 = $x and column2 = $(a + b)")
```
如果需要讓 `$` 作為字符串原始的意思，你需要另外一個 `$` 轉義它（用兩個連續的`$`：`$$`）。

- get_connection()
獲取一個可用的數據庫連接。如果想直接調用 DBAPI 接口，這個函數是很有用的。ORM 自己也是用的這樣的方式。在 離開`db_session`上下文或者數據庫事務回滾的時候，連接將被重置並歸還到連接池。

- disconnect()
關閉當前線程已開始的數據庫連接。

### Database 狀態統計
class Database
`Database` 對象保留了命令執行的統計結果。你可以查看哪個命令執行的最多花費了多少時間，以及其他很多參數。Pony 對每個線程保留分離的統計結果。如果你想看所有線程的總計結果，需要調用 `merge_local_stats()` 方法。

- local_stats
這是一個當前線程的 SQL 語句的字典。字典的鍵是 SQL 語句，值是 `QueryStat` 類型的一個對象。

- class QueryStat
這個類型有一系列的屬性，用來累積對應則值：
`avg_time`, `cache_count`, `db_count`, `max_time`, `merge`, `min_time`, `query_executed`, `sql`, `sum_time`.

- merge_local_stats()
這個方法合並當前線程的統計結果到全局的統計結果。你可以在 HTTP 請求結束的時候調用這個方法。

- global_stats
這是一個匯總了所有線程的 SQL 語句的字典。字典的鍵是 SQL 語句，值是 `QueryStat` 類型的一個對象。在訪問這個對象之前，你應該調用 `db.global_stats_lock`。

## 使用 Database 對象做原生 SQL 命令
使用 Pony 可以使用方便的給 SQL 命令傳遞參數。為了傳遞變量，你需要將 $ 放置在變量名之前。
```python
x = "John"
data = db.select("* from Person where name = $x")
```
當 Pony 在 SQL 命令中遇到這種變量，它會從當前上下文獲取變量的值（從 globals 和 locals）,或者從第二個變量傳遞的一個字典。上面的例子中，Pony 嘗試從變量 x 中獲取值給 $x，同時排除 SQL 註入風險。下面的例子展示了如何傳遞一個包含參數值的字典。
```python
data = db.select("* from Person where name = $x", {"x" : "Susan"})
```
這種傳遞參數到 SQL 命令的方式非常靈活，而且不只是傳遞一個值，還可以支持 Python 表達式。為了使用表達式，需要將表達式跟在 $ 符號後面。

```python
data = db.select("* from Person where name = $(x.lower()) and age > $(y + 2)")
```

[^identity-map-cache]: identity map cache
