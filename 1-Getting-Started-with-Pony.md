# 開始使用 Pony

使用下面的命令安裝 Pony
`pip install pony`
Pony 可以被安裝在 Python2 2.6以上版本，沒有別的擴展。

為了確認 Pony 安裝正常，打開 Python 交互器，輸入：
`>>> from pony.orm import *`
這樣將導入所有使用 Pony 所必須的類和函數（不是特別大）。實際上你也可以自己選擇導入哪些內容，但是我們建議剛開始的時候還是使用先`import *`

熟悉 Pony 的最好的方法是在交互器下好好玩玩。讓我們創建一個示例數據庫用來存放 `Person`類的實體，給它添加三個對象，然後寫一個查詢。

## 創建數據庫
在 Pony 中，對象的實體是存儲在數據庫中的，所以我們必須先創建一個數據庫，在交互器中輸入：
`>>> db = Database('sqlite', ':memory:')`
這個命令創建了對象和數據庫的連接，第一個參數指出了我們要使用的 DBMS ，目前 Pony 支持 4 種類型的數據庫：'sqlite', 'mysql', 'postgresql' 和 'oracle'。接下來的參數跟 DBMS 是關聯的；它指定了你要要通過 DB-API 模塊鏈接的數據庫。對於 sqlite，必須指定數據庫的創建位置，要麽是數據庫的名字，要麽是一個字符串`:memory:`，如果數據庫的未知是內存中，它將在 Python 交互器會話關閉的時候被刪除。

如果需要在文件中存儲的數據庫，可以使用下面的代碼替帶。
`>>> db = Database('sqlite', 'test_db.sqlite', create_db=True)`
這樣，如果數據庫文件不存在，將會自動創建一個新的。我們的例子中還是使用內存的數據庫。

## 定義實體對象
現在，讓我們創建兩個實體對象，Person 和 Car 。Person 有兩個屬性，name 和 age ，Car 有兩個屬性，make 和 model。這兩個對象是一對多的關系。在 Python 交互器中輸入如下代碼：
```python
>>> class Person(db.Entity):
...     name = Required(str)
...     age = Required(int)
...     cars = Set("Car")
...
>>> class Car(db.Entity):
...     make = Required(str)
...     model = Required(str)
...     owner = Required(Person)
...
>>>
```
我們創建的類是從 `db.Entity` 繼承來的，這意味著它們不是普通的類型，而是實例存儲在 `db`對應的數據庫中的實體對象。Pony 允許我們同時支持多個數據庫，但是每個實體只能屬於一個數據庫。

在 Person 中，我們有三個屬性，`name`，`age`和`cars`。`name`和`age`是必須的屬性，也就是說，它們是非 `None` 的，`name`是個字符串，`age`是個數字。 

其中 `cars` 屬性設置了 `Car` 類型，這是一條關系聲明。它能夠存儲一系列 `Car` 類型的實例， `"Car"` 作為字符串出現，是因為類型 `Car` 在那個時候還沒有定義。

`Car` 實體對象有三個必須的屬性。`make` 和 `model` 是字符串，`owner` 是一對多關系的另外一邊。Pony 中的關系總是需要被分別出現在關系兩邊的類型中的兩條語句所定義。

如果我們需要創建一個多對多關系，在關系兩頭的類型中，我們都要聲明 `Set` 屬性。Pony 會自動創建中間的關系表。

在 Python3 中 `str` 表示 unicode 字符串，Python2 中有兩種方式表示字符串，`str` 和 `unicode`。從 Pony Release 0.6 開始，無論是使用 `str` 還是 `unicode`，都表示 unicode 字符串。我們建議使用 `str` 作為字符串聲明，因為這樣在 Python3 中看起來自然一點。

如果你想在交互器中查看一個實例類型的定義，可以使用 `show()` 函數。傳遞一個一個實例類型給這個函數可以看到對應的定義。

```python
>>> show(Person)
class Person(Entity):
    id = PrimaryKey(int, auto=True)
    name = Required(str)
    age = Required(int)
    cars = Set(Car)
```
你可能註意到了，實例類型多了一個額外的叫 `id` 的屬性。為什麽會這樣？

每個實例都必須包含一個主鍵，用以區分不同的實例。由於我們沒有手動設置主鍵，所以它自動創建了一個。如果主鍵是自動創建的，那它就是一個名字叫 `id` 的數字格式。如果主鍵是手動創建的，那麽可以指定它的名字，類型也可以是數字或者字符串。Pony 也支持多主鍵。

當主鍵是自動創建時，它的 `auto` 屬性總是被定義為 `True`。也就是說，它的值會自動增加，使用了數據庫的計數器或者序列功能。

## 映射實例到數據庫
現在我們需要創建數據表以存儲對象的數據，我們可以通過調用 `Database` 對象的如下方法來實現：
`>>> db.generate_mapping(create_tables=True)`
參數`create_tables=True`標志了如果數據表不存在就調用 `CREATE TABLE` 明亮創建它們。

所有需要映射到數據庫的實例都必須在調用 `generate_mapping()` 之前聲明。

## 延後數據庫綁定

從 Pony release 0.5 開始，有一個額外的方法來指定數據庫參數。現在你可以先創建數據庫對象，然後就可以定義實例的屬性了，最後再將數據庫對象綁定到指定的數據庫。
```python
### module my_project.my_entities.py
from pony.orm import *

db = Database()
class Person(db.Entity):
    name = Required(str)
    age = Required(int)
    cars = Set("Car")

class Car(db.Entity):
    make = Required(str)
    model = Required(str)
    owner = Required(Person)
```
```python
### module my_project.my_settings.py
from my_project.my_entities import db

db.bind('sqlite', 'test_db.sqlite', create_db=True)
db.generate_mapping(create_tables=True)
```

這樣，你可以分離實例定義和將它綁定到指定數據庫的過程。這對測試會有幫助。

## 使用 debug 模式
Pony 允許你在屏幕上查看發送到數據庫的 SQL 命令（或者在 log 文件，如果是配置了的）。輸入如下命令可以打開這個模式：
`>>> sql_debug(True)`
如果這個命令是在 `generate_mapping()` 之前調用的，那麽當創建表時，你也能看到生成表的 SQL 語句。

Pony 默認將調試信息輸出到 stdout，如果你導入了 Python 的標準日志模塊，Pony 將會使用它替代 stdout。使用 Python 的標準模塊，你可以將調試信息保存到文件。

```python
import logging
logging.basicConfig(filename='pony.log', level=logging.INFO)
```
註，必須配置 `level=logging.INFO`，因為日志模塊默認的輸出等級是 WARNING ，Pony 默認使用 INFO 等級輸出信息。Pony 有兩個日志記錄者： `pony.orm.sql` 記錄發送到數據庫的 SQL 語句， `pony.orm` 記錄其他的。

## 創建實例對象並插入數據庫
現在，讓我們創建 5 個對象，描述 3 個人和 2 輛車，然後將這些信息存入數據庫。使用如下命令實現：

```python
>>> p1 = Person(name='John', age=20)
>>> p2 = Person(name='Mary', age=22)
>>> p3 = Person(name='Bob', age=30)
>>> c1 = Car(make='Toyota', model='Prius', owner=p2)
>>> c2 = Car(make='Ford', model='Explorer', owner=p3)
>>> commit()
```

Pony 並不是在對象創建的時候就就將它們存入數據庫的，而是只有當 `commit()` 命令執行的時候才會保存。如果在調用 `commit()` 命令之前開啟了 debug 模式，你將會看到 5 個 `INSERT` 命令將對象存入到了數據庫。

## 編寫查詢命令
現在我們的數據庫中存儲了 5 個對象，我們可以嘗試一些查詢語句。例如，如下命令就返回了一個年齡大於 20 的 person 的列表。
```pyhton
>>> select(p for p in Person if p.age > 20)
<pony.orm.core.Query at 0x105e74d10>
```
`select` 函數將 Python 的生成器翻譯為 SQL 查詢語句，返回一個 `Query` 類型的實例。當遍歷這個 query 的時候， SQL 查詢語句被發送了一次。獲取對象的列表的一個方法是對它進行一次切片操作 `[:]`。
```python
>>> select(p for p in Person if p.age > 20)[:]

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."age" > 20

[Person[2], Person[3]]
```
在返回結果中，你可以看到發送到數據庫的 SQL 查詢語句和提取的對象的列表。當我們 print 一個查詢結果時，對象的實例由實例類型的名字和方括號中的主鍵組成：`Person[2]`。

我們可以用過 `order_by` 方法對查詢結果排序。如果我們只想要結果的一部分，可以使用 Python 中的切片的方法。例如，我們讓 people 按照名字排序，提取前兩個對象，可以這麽寫：
```python
>>> select(p for p in Person).order_by(Person.name)[:2]

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
ORDER BY "p"."name"
LIMIT 2

[Person[3], Person[1]]
```
有時候在交互器模式下，我們想要以列表的形式查看對象所有屬性的值。這可以通過對查詢結果列表調用 `.show()`方法來實現。
```python
>>> select(p for p in Person).order_by(Person.name)[:2].show()

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
ORDER BY "p"."name"
LIMIT 2

id|name|age
--+----+---
3 |Bob |30
1 |John|20
```
`.show()` 方法不顯示“對多”屬性，因為那將需要調用額外的查詢，而且或許會很龐大。那就是為什麽之前你看不到下掛的 car 的信息。但是如果對象有“對一”屬性，那它將一並顯示。
```python
>>> Car.select().show()
id|make  |model   |owner
--+------+--------+---------
1 |Toyota|Prius   |Person[2]
2 |Ford  |Explorer|Person[3]
```
如果我們不需要對象的列表，但是需要遍歷結果集，可以直接使用 `for` 循環，不需要切片操作。
```python
>>> persons = select(p for p in Person if 'o' in p.name)
>>> for p in persons:
...     print p.name, p.age
...
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."name" LIKE '%o%'

John 20
Bob 30
```
在上面的例子中，我們獲取到所有的名字中帶有小寫字母 'o' 的人，然後將名字和年齡都打印出來。

查詢也不是必須要返回實例對象。例如我們可以獲取對象的列表。
```python
>>> persons = select(p for p in Person if 'o' in p.name)
>>> for p in persons:
...     print p.name, p.age
...
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."name" LIKE '%o%'

John 20
Bob 30
```
或者一個元組
```python
>>> select((p, count(p.cars)) for p in Person)[:]

SELECT "p"."id", COUNT(DISTINCT "car-1"."id")
FROM "Person" "p"
  LEFT JOIN "Car" "car-1"
    ON "p"."id" = "car-1"."owner"
GROUP BY "p"."id"

[(Person[1], 0), (Person[2], 1), (Person[3], 1)]
```
上面的例子中我們就獲取了一個由 person 和他擁有的 cars 的數量做成的元組的列表。

我們也可以運行集合查詢。以下是一個查詢最大用戶年齡的例子。
```python
>>> print max(p.age for p in Person)
SELECT MAX("p"."age")
FROM "Person" "p"

30
```
Pony 允許你編寫比目前示例中覆雜的多的查詢語句。請參考手冊的以下章節。

## 獲取對象
如果要通過主鍵查詢對象，可以在方括號中填寫主鍵。
```python
>>> p1 = Person[1]
>>> print p1.name
John
```
你可能會註意到，實際上並沒有給數據庫發送數據。那是因為這個對象已經在 session 緩存中存在了。緩存可以減少提交到數據庫的請求數量。
通過其他屬性獲取對象：
```python
>>> mary = Person.get(name='Mary')

SELECT "id", "name", "age"
FROM "Person"
WHERE "name" = ?
[u'Mary']

>>> print mary.age
22
```
這種情況下，即使對象已經被緩存，查詢依然會被發送到數據庫，因為 `name` 不是唯一性的鍵。數據庫 session 緩存只會在通過主鍵或唯一性鍵查詢時起作用。

可以通過傳遞實例對象到 `show()` 函數，用於顯示所屬實例類和屬性值。

```python
>>> show(mary)
instance of Person
id|name|age
--+----+---
2 |Mary|22
```

## 更新一個對象
```python
>>> mary.age += 1
>>> commit()
```
Pony 持續跟蹤對象屬性的變化。當 `commit()` 執行時，在此期間所有改變的對象都將同步到數據庫。Pony 只會更新改變了的屬性。

## db_session
當你使用 Python 交互器來操作時，你不用考慮數據庫會話的問題，因為 Pony 已經自動替你維護了。但是當你在應用中使用 Pony 時，所有的數據庫交互都需要基於數據庫會話。為了達到這個目的，你需要用修飾器 `@db_session` 包裝你的需要數據庫操作的函數。
```python
@db_session
def print_person_name(person_id):
    p = Person[person_id]
    print p.name
    # database session cache will be cleared automatically
    # database connection will be returned to the pool

@db_session
def add_car(person_id, make, model):
    Car(make=make, model=model, owner=Person[person_id])
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```
`@db_session` 修飾器在函數退出後執行幾個非常重要的操作：
- 如果函數拋出異常，執行回滾操作
- 如果沒有異常，數據被改變了，執行 commits
- 歸還數據庫連接到連接池
- 清空數據庫 session 緩存

即使一個函數僅僅是讀取數據而不會做任何改變，也應該使用`@db_session` 修飾器，為了將數據庫連接歸還到連接池。

實例對象只有在 `@db_session` 修飾下才有效。如果你需要只用這個對象渲染一個 HTML 模版，也許應該工作在 `db_session` 之下。

另外一個使用 `db_session` 的選擇是作為上下文管理器而不是修飾器。
```python
with db_session:
    p = Person(name='Kate', age=33)
    Car(make='Audi', model='R8', owner=p)
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```

## 手動編寫 SQL
如果你需要手動編寫 SQL ，你可以這樣做：
```python
>>> x = 25
>>> Person.select_by_sql('SELECT * FROM Person p WHERE p.age < $x')

SELECT * FROM Person p WHERE p.age < ?
[25]

[Person[1], Person[2]]
```
如果你想直接操作數據庫，避免使用實體對象，可以通過 `Database` 對象的 `select` 方法。
```python
>>> x = 20
>>> db.select('name FROM Person WHERE age > $x')
SELECT name FROM Person WHERE age > ?
[20]

[u'Mary', u'Bob']
```

## Pony 示例
相比於直接手動創建模型，從 Pony 已經做好的例子中導入類似的模型是非常簡單的。下例是一個簡單的在線商店的例子，你可以在 Pony 的網站上看到數據表的結構。https://editor.ponyorm.com/user/pony/eStore 
導入示例的方法：
`>>> from pony.orm.examples.estore import *`

在配置開始時，SQLite 數據庫需要創建必要的數據表。為了遷移它們以及示例數據，可以執行如下程序：
`>>> populate_database()`
這個函數將會創建對象，然後把他們放到數據庫中。
當對象都被創建了，你可以編寫一個查詢，例如，你可以找到你可以找到訪客最多的城市。
```python
>>> select((customer.country, count(customer))
...        for customer in Customer).order_by(-2).first()

SELECT "customer"."country", COUNT(DISTINCT "customer"."id")
FROM "Customer" "customer"
GROUP BY "customer"."country"
ORDER BY 2 DESC
LIMIT 1
```
這次我們選擇 country 對象的集合，通過第二列（訪客數量）反轉排序，拿出訪客數最高的 country 。

在 `pony.orm.examples.estore` 模塊中的 `test_queries()`函數，可以找到更多的查詢示例。 
