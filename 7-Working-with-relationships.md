# 關系的使用

在 Pony 中，一個實體對象可以和其他實體對象建立關系。每個關系有兩端，分別由兩個實體對象的屬性定義。

```python
class Person(db.Entity):
    cars = Set('Car')

class Car(db.Entity):
    owner = Optional(Person)
```
在上面的例子中，我們在 `Person` 和 `Car` 實體對象之間，使用屬性 `cars` 和 `owner` 定義了一個個一對多關系。讓我們給我們的實體對象增加一組數據屬性，然後嘗試是一些例子：
```python
from pony.orm import *

db = Database('sqlite', ':memory:')

class Person(db.Entity):
    name = Required(str)
    cars = Set('Car')

class Car(db.Entity):
    make = Required(str)
    model = Required(str)
    owner = Optional(Person)

db.generate_mapping(create_tables=True)
```
現在讓我們來創建 `Person` 和 `Car` 的實例：
```python
>>> p1 = Person(name='John')
>>> c1 = Car(make='Toyota', model='Camry')
>>> commit()
```
通常，在你的程序中，你不需要手動調用 `commit` 函數，因為 `db_session` 會自動調用。但是當你在交互器中工作的時候，因為沒有離開 `db_session` 的動作，所以如果需要將數據存儲到數據庫，就得手動提交。

## 關系的建立
剛剛我們建立了實例 `p1` 和 `c1`,但是他們之間沒有建立關系。然我們檢查一下關系屬性的值。
```python
>>> print c1.owner
None

>>> print p1.cars
CarSet([])
```
`cars` 屬性的值是空的。
現在讓我們建立這兩個實體對象之間的關系：
```python
>>> c1.owner = p1
```
如果我們現在打印關系屬性的值，我們會看到：
```python
>>> print c1.owner
Person[1]

>>> print p1.cars
CarSet([Car[1]])
```

當我們給 `Car` 的實例指定了所有者之後，`Person` 的關系屬性 `cars` 也同時改變了。

我們也可以在創建 `Car` 的實例的時候，通過指定關系屬性來建立關系。
```python
>>> p1 = Person(name='John')
>>> c1 = Car(make='Toyota', model='Camry', owner=p1)
```
在我們的例子中，屬性 `owner` 是可選的，我已我們可以在任何時候給它賦值， 在創建 `Car` 實例的時候，或者以後。

## 集合的操作
`Person` 的 `cars` 屬性被聲明為集合，因此我們可以使用集合的方式來操作： add、remove、in、len、clear。

你可以使用 `add()` 和 `remove()` 方法增加或移除關系。

```python
>>> p1.cars.remove(Car[1])
>>> print p1.cars
CarSet([])

>>> p1.cars.add(Car[1])
>>> print p1.cars
CarSet([Car[1]])
```
你可以檢查一個對象是否包含在集合中：
```python
>>> Car[1] in p1.cars
True
```

或者，確定一個對象不在這個集合中：
```python
>>> Car[1] not in p1.cars
False
```

獲取集合的大小：
```python
>>> len(p1.cars)
1
```

這裏有幾種方法讓你創建一個屬於特定的 `person` 實例的 `car`。

有一種選擇是使用 `create()` 方法：
```python
>>> p1.cars.create(model='Toyota', make='Prius')
>>> commit()
```

現在我們可以驗證， `Car` 的實例已經被添加到 `Person` 實例的 `cars` 集合屬性了：
```python
>>> print p1.cars
CarSet([Car[2], Car[1]])
>>> p1.cars.count()
2
```

你可以枚舉遍歷一個集合屬性：
```python
>>> for car in p1.cars:
...     print car.model
Toyota
Camry
```

## 屬性提升

在 Pony 中，集合屬性提供了屬性提升的能力：一個集合獲取她的元素的屬性。
```python
>>> show(Car)
class Car(Entity):
    id = PrimaryKey(int, auto=True)
    make = Required(str)
    model = Required(str)
    owner = Optional(Person)
>>> p1 = Person[1]
>>> print p1.cars.model
Multiset({u'Camry': 1, u'Prius': 1})
```
這裏，我們使用 `show()` 方法打印了實體對象類型的屬性，然後打印了 `cars` 關系屬性的 `model` 方法。`cars` 屬性擁有所有 `Car` 實力類型的屬性: `id`、 `make`、 `model` 和 `owner`。在 Pony 中，我們把這個叫做多重集合，是使用字典實現的。字典的鍵顯示了屬性的值 —— 例如我們的例子中的，“Camry” 和 “Prius” 。字典的值表示了集合中這些值出現的次數。
```python
>>> print p1.cars.make
Multiset({u'Toyota': 2})
```
`Person[1]` 有兩輛 Toyota 。
我們可以枚舉這個多重集合：
```python
>>> for m in p1.cars.make:
...     print m
...
Toyota
Toyota
```

### 多重集合
待決[TBD]

- 聚合函數
- 明確的
- 請求中的子請求

### 集合屬性的參數
集合屬性用來定義“對多”一邊的關系。他們可以用來定義一對多或多對多關系。例如：
```python
class Photo(db.Entity):
    tags = Set('Tag', lazy=True, table='Photo_to_Tag')

class Tag(db.Entity):
    photos = Set(Photo)
```
這裏，`tags` 和 `photos` 都是集合。
下面是能夠在集合屬性定義的時候指定的參數。

class Set
定義對多關系

- lazy
當訪問指定的集合中的元素的時候（檢查一個元素是否屬於集合，增加或刪除元素），Pony 總是將整個集合載入到 `db_session` 的緩存中。通常這能減少數據庫請求從而提高性能。但是如果你的集合特別大，你最好不要把它們載入到內存。設置 `lazy=True` 告訴 Pony 不能將集合載入到內存，而是每次都發送數據庫請求。默認情況 `lazy=False`。

- reverse
指定要綁定關系的實體對象的屬性的名字。當在兩個實體對象之間有多個關系的時候使用。

- table
這個參數只有在多對多關系中使用，讓你可以指定數據庫中關系中間表的名字。

- column
- columns
- reverse_column
- reverse_columns
這些參數僅用於多對多關系，允許你指定中間列的名字。`columns` 和 `reverse_columns` 參數接收一個列表，當實體對象使用組合鍵的時候使用。通常使用 `column` 或 `columns`，如果你不喜歡列的名字，可以在所有的關系屬性中使用。

- cascade_delete
布爾值，控制著關聯的對象的級聯刪除。默認值基於關系另一邊的定義。如果那邊是 `Optional` - 默認值是 `False`，如果那邊是 `Required`, 默認值是 `True`。

- nplus1_threshold
這個參數用來微調 N+1 問題的閾值。

### 集合屬性的方法
你可以像 Python 的集合那樣使用集合屬性，標準操作有 : `in`、 `not in`、 `len`：
class Set

- len()
返回集合中對象的個數。如果集合沒有被載入緩存，這個方法會現將集合載入緩存，然後返回對象的數量。只有當你之後想要遍歷枚舉這些對象並要將它們載入內存的時候才使用這個方法。如果你不需要將集合再入內存，你可以使用個 `count` 方法。
```python
>>> p1 = Person[1]
>>> Car[1] in p1.cars
True
>>> len(p1.cars)
2
```
還有一些你可以在集合屬性上調用的方法。
class Set
下面是一些你可以在“對多”關系屬性上調用的方法。

- add(item)
- add(iter)
增加實例到集合，建立兩個實體對象實例之間的雙向關系。
```python
photo = Photo[123]
photo.tags.add(Tag['Outdoors'])
```
現在 `Photo` 實體對象的主鍵為 123 的實例和 `Tag['Outdoors']` 實例建立了關系。`Tag['Outdoors']` 的  `photos` 屬性也包含了 `Photo123` 對象。

通過給 `add()` 方法傳遞一個列表，我們也可以一次性建立多個關系：
`photo.tags.add([Tag['Party'], Tag['New Year']])`

- remove(item)
- remove(iter)
在集合中移除一個或多個元素，這樣就斷開了實體對象實例間的關系。

- clear()
從集合中移除所有元素，同時實體對象間的關系也斷開了。

- id_empty()
如果實例沒有關聯的兌現，返回 `True`。如果實例至少有一條對象，返回 `False`。

- copy()
返回一個 Python 中的 `set` 對象，其中包含了給定集合的所有元素。

- count()
返回集合中對象的個數。這個方法不會將集合中的對象都載入到緩存，而是生成一個 SQL 請求，從數據庫中查詢對象的個數。如果你之後要使用集合對象（枚舉集合對象，使用對象的屬性），你可以使用 `len()` 方法。

- creat(**kwargs)
創建一個關聯的實體對象的實例，並建立和它的關系：
```python
new_tag = Photo[123].tags.create(name='New tag')
```
相當於：
```python
new_tag = Tag(name='New tag')
Photo[123].tags.add(new_tag)
```

- load()
將所有關聯的對象從數據庫中載入。

### 集合類的方法
這個方法可以使用實體對象類型定義，為不是實例。例如：
```python
from pony.orm import *
db = Database('sqlite', ':memory:')

class Photo(db.Entity):
    tags = Set('Tag')

class Tag(db.Entity):
    photos = Set(Photo)

db.generate_mapping(create_tables=True)
Photo.tags.drop_table() # drops the Photo-Tag intermediate table
```

class Set

- drop_table(with_all_data=False)
刪除為了創建多對多關系的中間表。如果表是空的，而且 `with_all_data=False`，這個方法拋出異常： `TableIsNotEmpty`，而且不會刪除任何東西。設置 `with_all_data=True`，允許你刪除非空的表。

### 集合的請求

從 release 0.6.1 開始，Pony 引入了關系屬性的查詢。

你可以在對多關系中使用 `select()`、 `Query.filter()`、 `Query.order_by()`、 `Query.page()`、 `Query.limit()`、 `Query.random()` 方法。`select` 和 `filter` 是一樣的。

下面我們列舉幾個使用這些方法的例子。我們將使用大學課程作為請求展示。已經定義了 Python 實體對象和關系表。

下面的例子選取了，group 1 中 gpa 大於 3 的所有學生。
```python
g = Group[101]
g.students.filter(lambda student: student.gpa > 3)[:]
```

這個請求用於顯示，group 1 中按名字排序，第二頁的學生：
`g.students.order_by(Student.name).page(2, pagesize=3)`

這個請求也可以這麽寫：
`g.students.order_by(lambda s: s.name).limit(3, offset=3)`

下面的請求返回 group 1 中隨機的兩個學生。
`g.students.random(2)`

接下來，這個請求返回了 `Student[1]` 第二學期選擇的課程，按名字排序的第一頁。

```python
s = Student[1]
s.courses.select(lambda c: c.semester == 2).order_by(Course.name).page(1)
```
