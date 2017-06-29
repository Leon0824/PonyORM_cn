# 請求

Pony 提供了一個非常方便的途徑，使用生成器語法去請求數據庫。Pony 讓程序員操作存儲在數據庫中的對象和操作內存中的對象一樣，使用原生的 Python 語法。這讓開發工作更為簡單。

## Pony ORM 中用來請求數據庫的函數

- select(gen[, globals[, locals])
這個方法把生成器語法轉化為 SQL 請求，並返回 `Query` 類型的實例。如果有必要，你可以在結果上使用任何 `Query` 的方法，例如，`Query.order_by()` 或 `Query.count()` 。如果你只是想獲取一個對象的列表，你可以使用枚舉遍歷結果或者使用完整的切片。
```python
for p in select(p for p in Product):
    print p.name, p.price

prod_list = select(p for p in Product)[:]
```

`select()` 函數也可以返回一個單獨的屬性的列表，或者元組的列表：
```python
select(p.name for p in Product)

select((p1, p2) for p1 in Product
                for p2 in Product if p1.name == p2.name and p1 != p2)

select((p.name, count(p.orders)) for p in Product)
```
你可以傳遞 `globals` 和 `locals` 字典，將它們用作 `global` 和 `local` 名字空間。
你也可以將 `select` 用於關系屬性，請看[例子]()。

- get(gen[, globals[, locals])
從數據庫中提取一個實體對象的實例。如果指定參數的對象存在，返回這個對象，如果對象不存在，返回 `None`。如果滿足條件的對象有多個，會拋出異常：
`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：
`get(o for o in Order if o.id == 123)`
`Query.get()` 也會生成相同的請求：
`select(o for o in Order if o.id == 123).get()`

- left_join(gen[, globals[, locals])
左連接的查詢結果總是包含 "left" 表中的結果，即使在 "right" 表中沒有找到任何匹配項。
假如我們需要計算每位顧客的賬單的金額。讓我們使用 Pony 做這個例子。
```python
from pony.orm.examples.estore import *
populate_database()

select((c, count(o)) for c in Customer for o in c.orders)[:]
```
這將會被轉換為如下 SQL 語句：
```sql
SELECT "c"."id", COUNT(DISTINCT "o"."id")
FROM "Customer" "c", "Order" "o"
WHERE "c"."id" = "o"."customer"
GROUP BY "c"."id"
```
返回如下結果：
```python
[(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1)]
```
 但是沒有賬單的顧客將不會被這個請求選中，因為條件是 `WHERE "c"."id" = "o"."customer"` ，不會在賬單表中找到匹配的記錄。為了得到所有顧客的列表，我們應該使用 `left_join` 函數：
 `left_join((c, count(o)) for c in Customer for o in c.orders)[:]`
 ```sql
 SELECT "c"."id", COUNT(DISTINCT "o"."id")
FROM "Customer" "c"
  LEFT JOIN "Order" "o"
    ON "c"."id" = "o"."customer"
GROUP BY "c"."id"
 ```
 現在我們得到了所有顧客的列表，包含沒有賬單的顧客。
 ```python
 [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1), (Customer[5], 0)]
 ```
 我們也知道，大多數情況 Pony 能夠自動判斷 LEFT JOIN 是必須的，例如，相同的請求我們可以已這種方式來寫。
 `select((c, count(c.orders)) for c in Customer)[:]`
 ```python
 SELECT "c"."id", COUNT(DISTINCT "order-1"."id")
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
GROUP BY "c"."id"
 ```
- count(gen)
返回符合查詢條件的對象的個數，例如：
`count(c for c in Customer if len(c.orders) > 2)`
這個請求將會被轉換為如下 SQL 語句：
```sql
SELECT COUNT(*)
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
GROUP BY "c"."id"
HAVING COUNT(DISTINCT "order-1"."id") > 2
```
`Query.count()` 方法也會生成等價的請求：
```python
select(c for c in Customer if len(c.orders) > 2).count()
```
- min(gen)
返回數據庫中最小的值，請求會返回一個單獨的屬性的值。
`min(p.price for p in Product)`
`Query.min()` 方法也會生成等價的請求：
`select(p.price for p in Product).min()`

- max(gen)
- 返回數據庫中最大的值，請求會返回一個單獨的屬性的值。
`max(p.price for p in Product)`
`Query.max()` 方法也會生成等價的請求：
`select(p.price for p in Product).max()`

- sum(gen)
返回被選中的屬性的值的和：
`sum(o.total_price for o in Order)`
`Query.sum()` 方法也會生成等價的請求：
`select(o.total_price for o in Order).sum()`
如果沒有匹配到元素，`sum` 方法返回 0。

- avg(gen)
返回被選中的屬性的值的平均值：
`avg(o.total_price for o in Order)`
`Query.avg()` 方法也會生成等價的請求：
`select(o.total_price for o in Order).avg()`

- exists(gen[, globals[, locals])
如果至少一條指定條件的對象存在，返回 `True`，其他情況返回 `False`。
`exists(o for o in Order if o.date_delivered is None)`

- distinct(gen)
當你想要強制的返回唯一的請求，可以使用  `distinct()` 函數。
`distinct(o.date_shipped for o in Order)`
通常情況下這是非必須的，因為 Pony 會聰明的自動添加 DISTINCT 。查看下一節 Automatic DISTINCT 獲取更多信息。
另外一個使用 `distinct()` 的地方是和 `sum` 一起用。你可以這麽寫：
```python
select(sum(distinct(x.val)) for x in X)
```
會生成如下 SQL：
```sql
SELECT SUM(DISTINCT x.val)
FROM X x
```
但是很少在生產中使用。

- desc()
- desc(attribute)
在函數 `Query.order_by()` 內部調用，為了實現倒序排列。


## Query 對象的方法
class Query

- [start:end]
- limit(limit, offset=None)
這個方法用來限定從數據庫中選中的實例的數量。下面的例子選中了前十個元素。
`select(c for c in Customer).order_by(Customer.name).limit(10)`
會生成如下 SQL：
```python
SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
FROM "Customer" "c"
ORDER BY "c"."name"
LIMIT 10
```
Python 切片操作也可以達到相同的目的：
`select(c for c in Customer).order_by(Customer.name)[:10]`
如果我們需要選中指定偏移量的實例，可以使用第二個參數：
`select(c for c in Customer).order_by(Customer.name).limit(10, 20)`
或者使用切片操作：
`select(c for c in Customer).order_by(Customer.name)[20:30]`
會生成如下 SQL：
```python
SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
FROM "Customer" "c"
ORDER BY "c"."name"
LIMIT 10 OFFSET 20
```
`page()` 方法也會實現相同的目的。

- avg()
返回選中的屬性的平均值
`select(o.total_price for o in Order).avg()`
`avg()` 函數做了相同的工作。

- count()
返回滿足條件的對象的個數
`select(c for c in Customer if len(c.orders) > 2).count()`
`count()` 函數做了相同的工作。

- distinct()
強制開啟 DISTINCT 請求：
`select(c.name for c in Customer).distinct()`
通常情況下這是非必須的，因為 Pony 會聰明的自動添加 DISTINCT 。查看下一節 Automatic DISTINCT 獲取更多信息。`distinct()` 函數做了相同的工作。

- exists()
如果至少一條指定條件的對象存在，返回 `True`，其他情況返回 `False`。
`select(c for c in Customer if len(c.cart_items) > 10).exists()`
這個請求生成了如下 SQL:
```python
SELECT "c"."id"
FROM "Customer" "c"
  LEFT JOIN "CartItem" "cartitem-1"
    ON "c"."id" = "cartitem-1"."customer"
GROUP BY "c"."id"
HAVING COUNT(DISTINCT "cartitem-1"."id") > 20
LIMIT 1
```

- filter(lambda[, globals[, locals])
- filter(str)
`Query` 對象的 `filter()` 方法，是為了對查詢結果進行過濾。做為`filter()` 方法參數傳遞的過濾條件將會被翻譯成 WHERE 區的 SQL 請求。
`filter()` 參數的個數應該匹配請求結果。`filter()`可以用 lambda 表達式作為過濾條件：
```python
q = select(p for p in Product)
q2 = q.filter(lambda x: x.price > 100)

q = select((p.name, p.price) for p in Product)
q2 = q.filter(lambda n, p: n.name.startswith("A") and p > 100)
```
`filter()` 方法也可以傳遞一個表達式的字符串：
```python
q = select(p for p in Product)
x = 100
q2 = q.filter("p.price > x")
```
另外一種過濾請求結果的方式是傳遞指定參數的名字：
```python
q = select(p for p in Product)
q2 = q.filter(price=100, name="iPod")
```

- first()
返回選中的結果的第一個對象，當沒有對象的時候返回 `None`：
`select(p for p in Product if p.price > 100).first()`

- for_update(nowait=False)
有些時候需要鎖定數據庫中的對象，保證別的事務不是在相同的時候修改相同的對象。在數據庫中，這種鎖使用過使用 SELECT FOR UPDATE 實現的。如果要在 Pony 中使用這樣的鎖，可以使用 `for_update` 方法：
`select(p for p in Product if p.picture is None).for_update()[:]`
這個請求選中了沒有圖片的產品的實例並且鎖定了它們。鎖將會在提交或者回滾的時候解除。

- get()
從數據庫中講一個實例還原。如果對象存在返回滿足參數的對象，當沒有對象的時候返回 `None`，如果有多個對象滿足要求，拋出異常：`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：
`select(o for o in Order if o.id == 123).get()`
`get()` 函數做了相同的工作。

- max()
返回數據庫中最大的值，請求會返回一個單獨的屬性的值。
`select(o.date_shipped for o in Order).max()`
`max()` 函數也會生成等價的請求：

- min()
返回數據庫中最小的值，請求會返回一個單獨的屬性的值。
`select(o.date_shipped for o in Order).min()`
`min()` 函數也會生成等價的請求：

- order_by(attr1[, attr2, ...])
- order_by(pos1[, pos2, ...])
- order_by(lambda[, globals[, locals])
- order_by(str)
這個方法用來給請求結果排序。可用的用法有：
 - 基於實體對象屬性
`select(o for o in Order).order_by(Order.customer, Order.date_created)`
對任何屬性使用 `desc()` 函數或 `desc()` 方法，可以實現倒排序：
`select(o for o in Order).order_by(Order.date_created.desc())`
也等價於：
`select(o for o in Order).order_by(desc(Order.date_created))`
 - 基於請求結構變量的位置
`select((o.customer.name, o.total_price) for o in Order).order_by(-2, 1)`
負號意為倒排序。在例子中，我們按照總價倒序顧客名正序的方式進行了排序。
 - 基於 lambda
`select(o for o in Order).order_by(lambda o: (o.customer.name, o.date_shipped))`
如果 lambda 有一個參數（在這個例子中是 `o`），那麽 `o` 傳遞了 `select` 的結果。如果你指定了一個沒有參數的 lambda，那麽 `query` 中的所有名字都能訪問：
`select(o.total_price for o in Order).order_by(lambda: o.customer.id)`
 - 基於字符串
這個方法和上一個方法類似，但是你將 lambda 的內容作為一個字符串傳遞了。
`select(o.total_price for o in Order).order_by(lambda: o.customer.id)`

- page(pagenum, pagesize=10)
當你想在多個頁面展示請求結果，你會用到分頁。頁碼從 1  開始計數。這個方法返回一個切片 [start:stop] ,其中 `start = (pagenum - 1) * pagesize, stop = pagenum * pagesize`。

- prefetch()
允許你指定，哪些關聯的對象和屬性需要一起被載入。
通常沒必要 prefetch 關聯的對象。當你在 `@db_session` 下操作請求結果的時候，Pony 獲取所有你需要的關聯的對象。Pony 使用最有效率的方法將關聯的對象從數據庫中獲取出來。避免了 N + 1 問題
。
當你使用 Flask 框架的時候，就推薦使用這種方法，將`@db_session`修飾器放在函數上面，和 `app.route` 的修飾器一起。
```python
@app.route('/index')
@db_session
def index():
    ...
    objects = select(...)
    ...
    return render_template('template.html', objects=objects)
```
或者，一樣有效的，使用 `db_session` 包裝 wsgi 應用：
`app.wsgi_app = db_session(app.wsgi_app)`
有些情況之下，你需要傳遞選中的實例和關聯的對象到 `db_session` 外，那麽你可以使用這個方法。要不然，當你嘗試在 `db_session` 外訪問關聯的對象的時候會得到異常 ： `DatabaseSessionIsOver`，例如：
`DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over`
關於 `db_session` 的更多工作可以在這裏找到。
你可以指定實體對象或者和屬性作為對象。當你指定一個實體對象，那麽“對以”關系和非懶惰屬性將被預存取。“對多”關系只有在指定的時候才會預存取。
如果你指定了一個屬性，那麽只有這個屬性會被預存。你可以指定屬性鏈，例如， order.customer.address。預存取是遞歸的 —— 對每個指定的對象使用。
例如：
`from pony.orm.examples.presentation import *`

載入學生對象，不進行預存取：
`students = select(s for s in Student)[:]`
同時載入學生、組和部門：
```python
students = select(s for s in Student).prefetch(Group, Department)[:]

for s in students: # no additional query to the DB will be sent
    print s.name, s.group.major, s.group.dept.name
```
跟上面的一樣，但是指定屬性而不是實體對象。
```python
students = select(s for s in Student).prefetch(Student.group, Group.dept)[:]

for s in students: # no additional query to the DB will be sent
    print s.name, s.group.major, s.group.dept.name
```
載入學生和關聯的課程。（多對多關系）
```python
students = select(s for s in Student).prefetch(Student.courses)

for s in students:
    print s.name
    for c in s.courses: # no additional query to the DB will be sent
        print c.name
```

- random(limit)
從數據庫中選擇 `limit` 個隨機對象。這個方法會轉義為 `ORDER BY RANDOM()` SQL 語句。實體對象的方法 `select_random()` 提供了更好的性能，雖然不允許指定查詢條件。

- show()
將請求結果打印到控制台。結果會被格式化為表格的形式。這個方法不會顯示對多屬性，因為那可能會需要額外的查詢而且可能很大。

- sum()
返回選擇的元素的和。只可以用開請求數字類型的屬性。例如：
`select(o.total_price for o in Order).sum()`
如果查詢結果沒有元素，那麽請求結果將會是 0。

- without_distinct()
默認情況下，Pony 嘗試避免重覆的請求結果，當必要的時候，智能的添加 `DISTINCT` SQL 關鍵詞。如果你不需要添加 `DISTINCT`，允許獲取可能重覆的值，那麽你可以使用這個方法：
`select(p.name for p in Person).without_distinct().order_by(Person.name)`
Pony 從 Release 0.6 開始，`without_distinct()` 返回一個請求結果，不是一個新的請求實例。

## Automatic DISTINCT
Pony 嘗試避免重覆的請求結果，當必要的時候，智能的添加 `DISTINCT` SQL 關鍵詞，因為有用的重覆結果很少見。當人們想按照指定規則恢覆對象的時候，通常不會希望相同的對象返回多個。而且，避免重覆讓讓查詢結果更可控：你不需要再去過濾相同的結果。

Pony 只有在可能有重覆內容的時候才添加 `DISTINCT`。讓我們來看一些例子。

1. 按條件恢覆對象
`select(p for p in Person if p.age > 20 and p.name == "John")`
在這個例子中，請求不會出現沖虛，因為結果中包含主鍵。簡單的重覆在這裏不可能發生，所以沒必要添加 `DISTINCT` 關鍵詞，Pony 也沒有添加：
```sql
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."age" > 20
  AND "p"."name" = 'John'
```
2. 恢覆對象屬性
`select(p.name for p in Person)`
這個查詢結果返回的不是對象而是它的屬性，請求結果可能是重覆的，所以 Pony 添加了 `DISTINCT`。
```sql
SELECT DISTINCT "p"."name"
FROM "Person" "p"
```
這種請求的結果通常用在下拉列表中，一般是不希望有重覆的。也可以很簡單的切換到有重覆的查詢結果。
如果你想統計名字相同的人的個數，建議你這麽做：
`select((p.name, count(p)) for p in Person)`
如果非常有必要獲取人們的名字，包含重覆的，你可以使用 ` Query.without_distinct()` 方法：
`select(p.name for p in Person).without_distinct()`

3. 並表查詢對象
`select(p for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))`
這個請求結果可能是有重覆的，所以 Pony 添加了 `DISTINCT`。
```sql
SELECT DISTINCT "p"."id", "p"."name", "p"."age"
FROM "Person" "p", "Car" "c"
WHERE "c"."make" IN ('Toyota', 'Honda')
  AND "p"."id" = "c"."owner"
```
不使用 `DISTINCT`數據是可能重覆的，因為這個請求使用了兩個表（人和 車），但是只有一個表在選擇條件下。請求結果只返回人（不包含車），因此這是一個經典的，讓人不滿的，結果中獲取多個重覆的人。我們相信，去掉重覆的內容，結果看起來更明了。
但是，如果有些原因不需要執行 desirable ，你總可以給請求中添加 `.without_disctinct()`。
```python
select(p for p in Person for c in p.cars
         if c.make in ("Toyota", "Honda")).without_distinct()
```
在請求結果中包含車和它的所有者，這種情況用戶應該更想要看到重覆的人的對象。這種情況 Pony 的請求是不同的。
`select((p, c) for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))`
Pony 沒有添加 `DISTINCT`。
總結一下：
    1. 原則上，“默認請求不返回重覆的結果”，這很容易理解，也不意外。
    2. 這種行為時大多數人在大多數情況下需要的。
    3. 請求不想要 duplicates 的時候， Pony 不添加 DISTINCT。
    4. 使用 `without_distinct()` 可以消除 Pony 使用 DISTINCT。


## 請求中可以使用的函數
這裏是能夠在生成器中使用的函數的列表：
min、 max、 avg、 sum、 count、 len、 concat、 abs、 random、 select、 exists。

例子：
`select(avg(c.orders.total_price) for c in Customer)[:]`
```sql
SELECT AVG("order-1"."total_price")
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
```
```sql
select(o for o in Order if o.customer in
       select(c for c in Customer if c.name.startswith('A')))[:]
```
```sql
SELECT "o"."id", "o"."state", "o"."date_created", "o"."date_shipped",
       "o"."date_delivered", "o"."total_price", "o"."customer"
FROM "Order" "o"
WHERE "o"."customer" IN (
    SELECT "c"."id"
    FROM "Customer" "c"
    WHERE "c"."name" LIKE 'A%'
    )
```
