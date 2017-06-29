# 實體對象的實例的使用

## 創建一個實體對象的實例
Pony 中創建一個實體對象的實例跟創建一個 Python 對象是一樣的：
```python
customer1 = Customer(login="John", password="***",
                     name="John", email="john@google.com")
```
當創建一個 Pony 對象的時候，所有的參數需要以關鍵詞參數的形式指定。如果屬性有默認值的，可以省略。

所有創建的實例都屬於當前數據庫 session。在一些對象-關系映射器中，你需要調用對象的 `save()` 方法來保存它。這是不方便的，作為程序員，必須跟蹤有哪些對象被創建或更新了，必須要記得在每個對象上調用 `save()` 方法。

pony 自動跟蹤有哪些對象被創建或更新了，當當前 `db_session` 結束的時候自動保存。如果你想在 `db_session` 結束前保存新建的對象，可以使用 `flush()` 或 `commit()` 函數。

## 從數據庫中載入對象

### 通過主鍵獲取對象
最簡單的情況是，當我們使用主鍵恢覆一個對象的時候。在 Pony 中實現這個功能，用戶只需要將主鍵放在類名後面的方括號中。例如，提取主鍵值是 123 的用戶，我們可以寫為：
`customer1 = Customer[123]`
對於組合鍵的對象語法也一樣，我們需要將組合鍵的元素一一列出，使用逗號分隔，順序是實體對象類型定義的時候屬性的順序。
`order_item = OrderItem[order1, product1]`
如果指定主鍵的對象不存在，Pony 拋出 `ObjectNotFound` 異常。

### 使用唯一性組合鍵獲取唯一對象
當我們不使用主鍵，而是使用其他屬性的結合來提取對象的時候，我們可以在實體對象上使用 `get` 方法。在大多數情況下，我們使用這種方法是通過次位的唯一性屬性搜索對象，其實也可以使用其他屬性的結合來搜索對象。作為 `get` 方法的參數，我們指定屬性的名字和值。例如，我們想要恢覆 name 是 “Product 1” 的 product 對象，我們認為數據庫中只有一個這樣的對象，那麼可以這麼寫：
`product1 = Product.get(name="Product1")`

如果沒有找到任何對象，`get` 返回 `None`。如果找到了多個對象，會拋出 `MultipleObjectsFoundError` 異常。

當想要在對象不存在的情況下獲得 `None` 而不是 `ObjectNotFound` 異常時，通過主鍵調用 `get`方法是可以的。

`get()` 方法也可以接收一個 lambda 表達式作為唯一的位置型參數，跟之後要討論的 `select()` 方法類似，但是返回結果是實體對象，而不是 `Query` 類型的對象。

### 獲取多個對象
為了從數據庫恢覆多個對象，我們使用每個實體對象都有的 `select()` 方法。他的參數是一個 lambda 表達式，接收一個數據庫對象實例的別名作為參數。在表達式中，我們可以編寫我們想要檢索的條件。例如，我們想要找出 products 中 price 大於 100 的，我們可以這麼寫：
`products = Product.select(lambda p: p.price > 100)`
這個 lambda 表達式將不會被 Python 執行，而是被轉化為 SQL 語句：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > 100
```
`select()` 返回 Query 類型的對象。如果你在這個對象上使用疊代，SQL 語句將被發送到數據庫，你將得到一個實體對象的數組。例如，這就是我們怎樣打印所有符合要求的的產品的名字和價格。
```python
for p in Product.select(lambda p: p.price > 100):
    print p.name, p.price
```
如果我們不想完全疊代一個查詢結果，僅僅是需要一個對象的列表，我們可以這樣做：
`product_list = Product.select(lambda p: p.price > 100)[:]`
這裏，我們從結果中你獲取了一個完整切片，這和將結果轉為列表是等價的。
`product_list = list(Product.select(lambda p: p.price > 100))`

###  向語句中傳遞參數
在 lambda 表達式中，可以使用之前聲明的參數。在這種情況下，查詢中會將這些變量的值傳遞給參數。Pony 查詢語句的一個重要的優勢是提供完全的 SQL 註入保護，像這種外面聲明的參數的值都將被適當的轉換。

例如，我們想要找出price 高於 x  的 products，我們可以簡單的寫成。
```python
x = 100
products = Product.select(lambda p: p.price > x)
```
這個 SQL 查詢將被生成為：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > ?
```
`x` 的值將會傳遞給 SQL 語句，而且完全避免 SQL 註入風險。

### 查詢結果排序
如果我們要將對象按照一定的順序排序，我們可以使用 `Query` 對象的 `order_by()` 方法。如果我們想倒序顯示價格高於 100 的產品的名字和價格，我們可以這麼做：
`Product.select(lambda p: p.price > 100).order_by(desc(Product.price))`
`Query` 對象的方法會修改發往數據庫的 SQL 語句。在如上的例子中，將會生成下面的 SQL：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > 100
ORDER BY "p"."price" DESC
```
`order_by()` 方法也可以接收一個 lambda 表達式作為參數：
`Product.select(lambda p: p.price > 100).order_by(lambda p: desc(p.price))`

`order_by()` 方法中使用 lambda 表達式可以實現更高級的排序方法。例如，我們要將客戶按照客戶的賬單總價倒序排列：
`Customer.select().order_by(lambda c: desc(sum(c.orders.total_price)))`

為了使用多個屬性對結果排序，我們用逗號分割它們。例如，如果我們希望按照產品價格排序，如果價格相同，則使用產品名字母順序排序，我們可以這麼做。
`Product.select(lambda p: p.price > 100).order_by(desc(Product.price), Product.name)`

相同的請求，但是使用 lambda 實現，如下：
`Product.select(lambda p: p.price > 100).order_by(lambda p: (desc(p.price), p.name))`

註意，為了和 Python 的語法一致，如果希望從 lambda 返回多個元素，我們需要使用括號包裹它們。

### 限定選中的對象的個數

使用 Query 的 limit() 方法，或者更簡潔的 Python 的 slice 標記，可以限定請求的對象的數量。例如，這是我們如果和獲取最貴的是個產品的方法。
`Product.select().order_by(lambda p: desc(p.price))[:10]`
slice 的結果不是 Query 對象，而是實體對象實例的列表。

你也可以使用 Query.page() 方法作為請求結果分頁實現的簡單方法：
`Product.select().order_by(lambda p: desc(p.price)).page(1)`

### 反查關系
在 Pony 中你可以很簡單的反查關系：
```python
order = Order[123]
customer = order.customer
print customer.name
```
Pony 嘗試最小化發送到數據庫的請求的數量。在上面的例子中，如果 `Customer` 的對象已經在緩存中存在，Pony 將不會發送數據庫請求直接返回緩存中的對象。但是如果對象沒有被載入，Pony 也還是不會立即發送請求。而是先創建了一個 “種子” 對象。種子就是只有主鍵被初始化的對象。Pony 並不知道這個對象什麼時候會被使用，總是存在一個可能，只需要主鍵就足夠了。

在上面的例子中，Pony 在第三行執行的時候從數據庫獲取對象，即當我們訪問 name 屬性的時候。通過“種子”思想，Pony 獲得了極高的效率和解決了“N+1”問題，這些是其他映射器的軟肋。

在“對多”關系中反查也是可以的。例如，如果我們有一個 `Customer` 對象，我們想要遍歷他的賬單，我們可以這麼做：

```python
c = Customer[123]
for order in c.orders:
    print order.state, order.price
```

## 更新一個對象
當你給對象的屬性賦新值之後，不需要對更新的對象手動保存。修改將會在離開 `db_session` 區域的時候自動保存。
例如，為了給主鍵為 123 的對象數量加 10，可以執行如下代碼：
`Product[123].quantity += 10`
如果我們需要蓋面一個對象的多個方法，我們可以分開做：
```python
order = Order[123]
order.state = "Shipped"
order.date_shipped = datetime.now()
```
或者使用 `set()` 方法，只有一行：
```python
order = Order[123]
order.set(state="Shipped", date_shipped=datetime.now())
```
`set()` 方法在使用字典更新一個對象的多個屬性時很方便：
`order.set(**dict_with_new_values)`

如果你想在 `db_session` 結束前保存新建的對象，可以使用 `flush()` 或 `commit()` 函數。

在執行 `select()`、 `get()`、 `exists()`、 `execute()` 和 `commit()` 之前，Pony 總是自動增量的保存 `db_session` 的緩存到數據庫。

未來，Pony 將會支持大塊更新。那將允許在硬盤中直接更新大量對象而不需要將他們讀取到內存：
`update(p.set(price=price * 1.1) for p in Product if p.category.name == "T-Shirt")`

## 刪除一個對象
當你調用實體對象的實例的 `delect()` 方法時，這個對象將被標記為已刪除。在之後的提交之後，對象將在數據庫中被刪除。
例如，這就是我們怎樣刪除一個主鍵等於 123 的對象的方法。
`Order[123].delete()`

未來，Pony 將會支持大塊刪除。那將允許在硬盤中直接刪除大量對象而不需要將他們讀取到內存：
`delete(p for p in Product if p.category.name == "Floppy disk")`

### 關聯刪除
當 Pony 刪除一個實體對象的實例的時候，需要同時刪除和其他對象的關系。兩個對象的關系是由兩條關系屬性定義的。如果關系的另一邊聲明的是 `Set`，那麼我們只需要從集合中刪除那個對象就好了。如果另一邊聲明的是 `Optional`，那麼我們將它設置為 `None`，如果另一邊聲明的是 `Required`，我們不能直接將關系屬性改為 `None`。這種情況下，`Pony` 嘗試繼續刪除關聯的對象。這個默認的行為可以被 `cascade_delete` 屬性改寫。如果關系的另一邊是 `Required` ，這個屬性默認的值是 `True` ，其他類型的關系，默認是 `False`。

`True` 意味著 Pony 總是做關聯的刪除，即時另一邊定義的是 `Optional`。 `False` 意味著 Pony 總是不對這個關系做關聯的刪除。如果另一邊關系定義為 `Required`，而且 `cascade_delete=False` Pony 在嘗試刪除的時候會拋出異常
`ConstraintError`。

一對多關系中關聯刪除的例子。嘗試刪除和學生關聯的小組時，會拋出異常 `ConstraintError`：

```python
class Group(db.Entity):
    major = Required(str)
    items = Set("Student", cascade_delete=False)

class Student(db.Entity):
    name = Required(str)
    group = Required(Group)
```

一對一關系中關聯刪除的例子。當刪除 `Person` 的時候，會刪除關聯的 `Passport`：

```python
class Person(db.Entity):
    name = Required(str)
    passport = Optional("Passport", cascade_delete=True)

class Passport(db.Entity):
    number = Required(str)
    person = Required("Person")\
```

## 實體對象的方法
class Entity
- []
返回通過主鍵選定的實體對象的實例。如果沒有對象，拋出 `ObjectNotFound` 異常。例如:
`p = Product[123]`
對於使用覆合主鍵的對象，使用逗號分隔兩個主件。
```python
order_id = 123
product_id = 456
item = OrderItem[123, 456]
```
如果主鍵指定的實體對象在 `db_session` 中被載入，Pony 直接從緩存中返回數據，不再給數據庫發請求。

- describe()
返回一個描述實體對象的字符串。例如：
```python
>>> from pony.orm.examples.estore import *
>>> print OrderItem.describe()

class OrderItem(Entity):
    quantity = Required(int)
    price = Required(Decimal)
    order = Required(Order)
    product = Required(Product)
    PrimaryKey(order, product)
```

- drop_table(with_all_data = False)
刪除數據庫中與實體對象對應的表。如果 `with_all_data = False` 而且表不為空，這個方法會拋出 `TableIsNotEmpty` 異常，不會刪除任何東西。設置 `with_all_data = True` 可以讓你刪除任何數據表，即使不為空。

如果你要刪除多對多關系中的中間表，你可以使用實體對象類型的（不是實例的） `drop_table` 方法。

```python
class Product(db.Entity):
    tags = Set('Tag')

class Tag(db.Entity):
    products = Set(Product)

Product.tags.drop_table(with_all_data=True) # removes the intermediate table
```
- exists(lambda[, globals[, locals])¶
- exists(**kwargs)
如果滿足指定條件或屬性值的實例存在，返回`True`，反之，返回 `False`。例如：
```python
Product.exists(price=1000)

Product.exists(lambda p: p.price > 1000)
```
- get(lambda[, globals[, locals])
- get(**kwargs)

用於從數據庫中恢覆一個實體對象。如果滿足條件的對象存在，返回這個兌現個。如果沒有這個對象，返回 `None`。如果有多個對象滿足條件，拋出異常
`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：

```python
Product.get(price=1000)

Product.get(lambda p: p.name.startswith('A'))
```

- get_by_sql(sql, globals=None, locals=None)
- select_by_sql(sql, globals=None, locals=None)

如果你發現你不能用標準的 Pony 請求表達一次請求，你可以使用你自己的 SQL 請求，Pony 會根據返回結果建立實體對象的實例。當 Pony 獲得 SQL 請求的結果時，會分析數據庫遊標返回的字段的名字。如果你是用的請求語句是 `SELECT * ...` ，這樣可能也獲取到了足夠的的信息來構建實體對象的實例。你也可以給請求中傳遞參數，查看使用原始 SQL 章節獲取更多信息。

- get_for_update(lambda,[globals[, locals], nowait=False)
- get_for_update(**kwargs, nowait=False)

跟 `get()` 方法一樣，但是使用 `SELECT ... FOR UPDATE` SQL 請求鎖定的這行數據。如果設置了 `nowait=True`, 如果該行已經被鎖定，這個方法將會拋出異常。如果設置了 `nowait=False`,它會等待直到這行被釋放。

如果你需要對多行使用 `SELECT ... FOR UPDATE`，你可以使用 `Query` 對象的 `for_update()` 方法。

- load()
- load(args)

載入所有惰性非惰性的屬性，但是不包括還沒有從數據庫恢覆的屬性。如果一個屬性已經被載入過了，將不會再次載入。你可以指定需要載入的屬性的列表將他們全部載入，或者指定它的名字讓 Pony 只載入他們：

```python
obj.load(Person.biography, Person.some_other_field)
obj.load('biography', 'some_other_field')
```

- select()
- select(lambda[, globals[, locals])

選取滿足 lambda 指定的篩選條件的對象。如果沒有指定 lambda 將選中所有對象。

`select` 方法返回 `Query` 類型的實例。實體對象的實例將會在枚舉 `Query` 對象的時候一次性從數據庫中請求。例如：
`Product.select(lambda p: p.price > 100 and count(p.order_items) > 1)[:]`

上面的請求返回了所有價格高於 100，售出大於一次的所有產品。

- select_random(limit)
選中 `limit` 個隨機對象。這個方法使用的邏輯比 `ORDER BY RANDOM()` SQL 語句更高效。這個方法使用如下邏輯：
1. 確定表中的最大 id
2. 在範圍內（0，max_id）生成一些隨機的 id
3. 將這些隨機的 id 還原成對象。如果某個 id 的對象不存在（例如，被刪除了），嘗試另外一個 id

只要需要就持續重覆步驟 2-3，直到獲得足夠多的對象。

即使在數據很多的表中這個邏輯也不影響性能，但是這個方法也有一些限制：

- 主件必須是順序的整數類型
- 兩個 id 之間的間隙（已刪除的對象的個數）應該相對的小。

如果你的請求沒有任何特定的標準，你可以使用 `select_random`，如果需要其他限定，你可以使用 `Query` 對象的 `random()`方法。

## 實體對象實例的方法
class Entity

- get_pk()
返回對象的主鍵的值。
```python
>>> c = Customer[1]
>>> c.get_pk()
1
```
如果主鍵是組合鍵，這個方法返回一個包含主鍵值的元組。
```python
>>> oi = OrderItem[1,4]
>>> oi.get_pk()
(1, 4)
```

- delete()
刪除一個實體對象的實例。對象將被標記為刪除，當離開 `db_session` 或者當發送下一條數據庫請求之前，將會自動調用 `flush()` 提交到當前事務。

- set(**kwargs)
一次性設置對象的多個屬性：
```python
Customer[123].set(email='new@example.com', address='New address')
```
這個方法在使用字典更新一個對象的多個屬性時很方便:
```python
d = {'email': 'new@example.com', 'address': 'New address'}
Customer[123].set(**d)
```
- to_dict(only=None, exclude=None, with_collections=False, with_lazy=False, related_objects=False)

返回一個屬性的名字和值的字典。這個方法在將對象序列化為 JSON 或其他格式的時候很有用。
默認情況下，這個方法不包含集合（對多關系）和惰性屬性。如果一個屬性的值是實體對象實例，那麼只有主鍵會被加入到字典。

`only` - 如果你只想得到指定的屬性，可以使用這個參數。這個屬性可以作為第一個位置參數。你可以指定一個屬性名字的列表 `obj.to_dict(['id', 'name'])`, 用空格分開的字符串 `obj.to_dict('id name')`，或者用逗號分隔的字符串 `obj.to_dict('id, name')`。

`exclude` - 這個參數允許你排除指定的屬性。屬性的名字可以使用類似 `only` 中的指定方式。

`related_objects` - 默認情況下，所有關聯的對象將會顯示為主鍵的形式。如果 `related_objects=True`，與當前對象有關系的對象都將會被加入到對象的結果字典中，不包含主鍵。如果你原本打算遍歷關聯的對象然後遞歸調用 `to_dict()` 方法的，這是一個有用的選項。

`with_collections` - 默認情況下，結果字典不包含集合（對多關系）。如果你設置了這個參數為 `True`，那麼對多關系將會以列表的形式展示。如果 `related_objects=False`（默認情況），這些列表將會由關聯的實例的主鍵組成。如果 related_objects=True，這些列表將會以對象的列表展示。

`with_lazy` - 如果為 `True`，那麼惰性屬性（例如，BLBOBs 二進制塊和 使用 `lazy=True` 指定的屬性）將會被包含在結果字典中。

為了演示這個方法的使用，我們使用 Pony 附帶的 eStore 例子。讓我們獲取一個 id = 1 的用戶，然後將他轉換為字典。
```python
>>> from pony.orm.examples.estore import *
>>> c1 = Customer[1]
>>> c1.to_dict()

{'address': u'address 1',
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'password': u'***'}
```

如果我們不想序列化密碼屬性，我們可以用這種方式執行:

```python
>>> c1.to_dict(exclude='password')

{'address': u'address 1',
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith'}
```
如果你想排除多個屬性，你可以以列表的指定它們：`exclude=['id', 'password']`、 `exclude='id, password'`、 `exclude='id password'`。

你也可以通過 `only` 來指定需要序列化的屬性。

```python
>>> c1.to_dict(only=['id', 'name'])

{'id': 1, 'name': u'John Smith'}

>>> c1.to_dict('name email') # 'only' parameter as a positional argument

{'email': u'john@example.com', 'name': u'John Smith'}
```
默認情況下，集合不包含在結果字典中。如果你想要包含他們，你想要要指定 `with_collections=True`。也可以在 `only` 參數中指定集合屬性。

```python
>>> c1.to_dict(with_collections=True)

{'address': u'address 1',
'cart_items': [1, 2],
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'orders': [1, 2],
'password': u'***'}
```
默認情況下，所有關聯的對象（卡，賬單）顯示為它們的主鍵的列表。如果你想要得到關聯的對象的實例，你可以使用 `related_objects=True`:
```python
>>> c1.to_dict(with_collections=True, related_objects=True)

{'address': u'address 1',
'cart_items': [CartItem[1], CartItem[2]],
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'orders': [Order[1], Order[2]],
'password': u'***'}
```
如果你需要序列化多個實體對象實例，或者序列化一個關聯者別的對象的實例，你可以使用 `pony.orm.serialization` 模塊中的函數 `to_dict`。

- flush()
將對象的改變保存到數據庫中。通常 Pony 會自動保存改變，你不需要自己調用這個方法。當你想要獲取一個新增對象的主鍵，而主鍵是數據庫自動累加的情況下，你可以使用這個方法。

## 實體對象鉤子
有時候，你可能需要在實體對象的實例即將在數據庫中創建、更新或刪除時執行一個動作。為了達到這個目的，你可以使用實體對象的鉤子。你可以在實體對象定義的時候對如下函數做你自己的實現。

class Entity
- before_insert()
只有在新創建的對象插入到數據庫的時候被調用。

before_update()
只有在對象更新修改到數據庫的時候被調用。

before_delete()
只有在對象從數據庫刪除的時候被調用。

after_insert()
當數據庫中插入了新的行之後調用。

after_update()
當數據庫中更新了一行之後調用。

after_delete()
只有在對象從數據庫刪除的之後被調用。

```python
>>> class Message(db.Entity):
...     title = Required(str)
...     content = Required(str)
...     def before_insert(self):
...         print "Before insert!"

>>> m = Message(title='First message', content='Hello, world!')
>>> commit()
Before insert!
INSERT INTO "Message" ("title", "content") VALUES (?, ?)
[u'First message', u'Hello, world!']
```

## 序列化實體對象的實例

### 使用 pickle 做序列化
Pony 允許序列化實體對象實例、請求結果和集合。當你想要講實體對象的實例存儲到外部的緩存中時( 例如, memcache )。當 Pony 序列化實體對象的實例的時候，會保存除了集合之外的所有屬性，為了避免序列化一個超大的序列。如果你想要序列化一個集合屬性，你需要單獨序列化它。例如：
```python
>>> from pony.orm.examples.estore import *
>>> products = select(p for p in Product if p.price > 100)[:]
>>> products
[Product[1], Product[2], Product[6]]
>>> import cPickle
>>> pickled_data = cPickle.dumps(products)
```
現在我們可以將序列化之後的數據放到緩存，稍後，當我們再次需要實例的時候，我們可以反序列化它：
```python
>>> products = cPickle.loads(pickled_data)
>>> products
[Product[1], Product[2], Product[6]]
```
為了提升性能，你可以將對象序列化存儲到外部緩存。當你反序列化一個對象的時候，Pony 將他們放入當前 `db_session`，看起來就像它們是從數據庫中載入的。即使對象的當前狀態和數據庫中的不符，Pony 也不做檢查。

### 序列化為字典並轉為 JSON
另外一種序列化實體對象實例的方法是使用實體對象實例的 `to_dict()` 方法，或者使用 `pony.orm.serialization` 中的 `to_dict()` 和 `to_json()` 函數。實例的 `to_dict()` 方法返回對應對象的鍵值字典結構。有時候你需要序列化的不是實例自己，而是和實例關聯的對象。這種情況下你可以使用下面介紹的 `to_dict()` 函數。

- to_dict()
 這個函數用作將實體對象實例序列化為字典。它接收一個實體對象的實例或者任何返回實體對象實例的疊代器。這個函數返回一個多級字典，其中包含傳遞給函數的對象以及和她直接關聯的對象。這裏有一個結果字典的結構：

```python
{
    'entity_name': {
        primary_key_value: {
            attr: value,
            ...
        },
        ...
    },
    ...
}
```
讓我們使用在線商店的模型例子（ [ER diagram](https://editor.ponyorm.com/user/pony/eStore) ），打印 `to_dict()` 函數的結果。註意，我們需要單獨導入 `to_dict `。

```python
from pony.orm.examples.estore import *
from pony.orm.serialization import to_dict

print to_dict(Order[1])

{
    'Order': {
        1: {
            'id': 1,
            'state': u'DELIVERED',
            'date_created': datetime.datetime(2012, 10, 20, 15, 22)
            'date_shipped': datetime.datetime(2012, 10, 21, 11, 34),
            'date_delivered': datetime.datetime(2012, 10, 26, 17, 23),
            'total_price': Decimal('292.00'),
            'customer': 1,
            'items': ['1,1', '1,4'],
            }
        }
    },
    'Customer': {
        1: {
            'id': 1
            'email': u'john@example.com',
            'password': u'***',
            'name': u'John Smith',
            'country': u'USA',
            'address': u'address 1',
        }
    },
    'OrderItem': {
        '1,1': {
            'quantity': 1
            'price': Decimal('274.00'),
            'order': 1,
            'product': 1,
        },
        '1,4': {
            'quantity': 2
            'price': Decimal('9.98'),
            'order': 1,
            'product': 4,
        }
    }
}
```
上面的例子中，結果包含了序列化的 `Order[1]` 的實例，和直接關聯的對象 `Customer[1]`, `OrderItem[1, 1]` 和 `OrderItem[1, 4]`。

- to_json()

這個函數使用了 `to_dict()` 的輸出，將它的 JSON 形式返回。

## 使用原始 SQL
即使 Pony 幾乎能夠將任意的 Python 寫的條件轉為 SQL，有些時候還是需要原始 SQL ，例如，為了調用預置的程序或者使用指定的數據庫的方言特性。在這種情況下， Pony 允許用戶已原始 SQL 的方式編寫請求，將它們作為字符串放在 `select_by_sql()` 或 `get_by_sql()` 中。
`products = Product.select_by_sql("SELECT * FROM Products")`
不同於 `select()`, `select_by_sql` 方法不返回 `Query` 對象，而是實體對象的列表。

使用以下語法可以將參數傳遞給 SQL 字符串：“$name_variable” or “$(expression in Python)”

```python
x = 1000
y = 500
Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)")
```

當 Pony 碰到 SQL 中的參數的時候，會從當前框架中獲取變量的值（從 globals 和 locals），或者從以參數形式傳遞的字典中獲取。

```python
Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)",
                       globals={'x': 100}, locals={'y': 200})
```

跟隨在 $ 符號之後的變量和表達式，將會以參數的形式自動計算和傳遞到數據庫請求中，已做了 SQL 註入檢測。Pony 自動將請求字符串中的 $x 替換為 ”?”, “%S” 或者當前數據庫系統中使用的其他參數形式。

如果請求中自己用到 `$` （例如，系統表的名字），它必須被寫成兩個連接的 `$`，就像這樣 `$$`。

## 將對象存入數據庫

### 保存對象的順序
通常情況下，Pony 將對象按照創建和修改的順序存入數據庫。有些情況，Pony 可以重新設計 SQL INSERT 語句，如果保存對象的時候需要。讓我們考慮如下情況：
```python
from pony.orm import *

db = Database('sqlite', ':memory:')

class TeamMember(db.Entity):
    name = Required(str)
    team = Optional('Team')

class Team(db.Entity):
    name = Required(str)
    team_members = Set(TeamMember)

db.generate_mapping(create_tables=True)
sql_debug(True)

with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    team = Team(name='Tenacity', team_members=[john, mary])
```
在上面的例子中，我們創建了兩個團隊成員和一個團隊對象，將這兩個成員分派給這個團隊。團隊成員和團隊之間的關系是在團隊成員的表中聲明的一個字段。

```python
CREATE TABLE "Team" (
  "id" INTEGER PRIMARY KEY AUTOINCREMENT,
  "name" TEXT NOT NULL
)

CREATE TABLE "TeamMember" (
  "id" INTEGER PRIMARY KEY AUTOINCREMENT,
  "name" TEXT NOT NULL,
  "team" INTEGER REFERENCES "Team" ("id")
)
```
當 Pony 創建了 `john`、 `mary` 和 `team` 對象，它明白必須要重排 SQL INSERT 語句的順序，首先在數據庫中創建 `Team` 對象的實例，那樣才能在團隊成員的字段中保存團隊的 id。
```python
INSERT INTO "Team" ("name") VALUES (?)
[u'Tenacity']

INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
[u'John', 1]

INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
[u'Mary', 1]
```

### 保存對象的循環鏈
現在讓我們設想，我們想要給團隊設置隊長。為了達到這個目的，我們需要給我們的實體對象增加一對屬性： `Team.captain` 和反轉的 `TeamMember.captain_of`
```python
class TeamMember(db.Entity):
    name = Required(str)
    team = Optional('Team')
    captain_of = Optional('Team')

class Team(db.Entity):
    name = Required(str)
    team_members = Set(TeamMember)
    captain = Optional(TeamMember, reverse='captain_of')
```
創建實體對象的實例並指定隊長的代碼如下：
```python
with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    team = Team(name='Tenacity', team_members=[john, mary], captain=mary)
```
當 Pony 執行上面的代碼的時候，會拋出異常：
`pony.orm.core.CommitException: Cannot save cyclic chain: TeamMember -> Team -> TeamMember`

為什麼會這樣？讓我們看看。Pony 看到要保存 `john` 和 `mary` 對象到數據庫，必須知道團隊的 id，然後就嘗試重排 insert 語句。但是當保存 `team` 對象的時候需要指定 `captain` 屬性，那又需要知道 `mary` 對象的屬性。這種情況下， Pony 不能解決這個循環鏈，然後拋出了異常。

為了保存這樣一個循環鏈，你需要添加 `flush()` 命令幫助 Pony：
```python
with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    flush() # saves objects created by this moment in the database
    team = Team(name='Tenacity', team_members=[john, mary], captain=mary)
```
這種情況下，Pony 將會先將 `john` 和 `mary` 對象保存到數據庫。之後使用 SQL UPDATE 語句建立和 `team` 之間的關系。

```python
INSERT INTO "TeamMember" ("name") VALUES (?)
[u'John']

INSERT INTO "TeamMember" ("name") VALUES (?)
[u'Mary']

INSERT INTO "Team" ("name", "captain") VALUES (?, ?)
[u'Tenacity', 2]

UPDATE "TeamMember"
SET "team" = ?
WHERE "id" = ?
[1, 2]

UPDATE "TeamMember"
SET "team" = ?
WHERE "id" = ?
[1, 1]
```
