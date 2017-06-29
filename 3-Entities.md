# 實體對象

實體對象就是對象的狀態存儲在數據庫中的 Python 類型。每個實體對象的實例對應數據庫的表中的一行。實體對象一般都代表著真實世界的對象。（例如，客戶，產品）

Pony 也提供一個實體對象關系表格編輯器，可以用來創建 Python 實體對象的定義。

在創建實體對象的實例之前，你需要先映射實體對象到數據表。Pony 能夠將實體對象映射到已經存在的表或者創建新的表。當映射生成之後，你就可以查詢數據或者創建新的實例。

## 定義一個實體對象
每個實體對象屬於一個數據庫。這就是為什麽在定義實體對象之前，需要創建一個 `Database` 類型的對象：
```python
from pony.orm import *

db = Database("sqlite", "database.sqlite", create_db=True)

class MyEntity(db.Entity):
    attr1 = Required(str)
```
Pony 的 Database 對象有一個 `Entity` 屬性，這個屬性是需要存儲在數據庫的實體對象的必須繼承的基類。

## Entity 屬性
Entity 的屬性就像類的屬性一樣，通過語法 `attr_name = kind(type)`定義。

```python
class Customer(db.Entity):
    name = Required(str)
    picture = Optional(buffer)
```
你也可以在屬性類型的括號中指定其他類型。
我們將會在之後的章節中詳細的討論。
Pony 有如下幾種屬性：

- Required
- Optional
- PrimaryKey
- Set

### Required 和 Optional
基本上大部分的屬性，都是 `Required` 和 `Optional` 類型。如果屬性是被定義為 `Required`，那麽任何時候都必須值，`Optional` 屬性的值可以是空的。

### Optional 字符串屬性
如果這個屬性沒有賦予值，大部分情況下默認值是 `None`。但是當一個屬性是字符串時，默認值是空字符串。在數據庫中使用空字符串比使用 `NULL` 更實用。大部分框架也都是這麽做的。空字符串也能提高搜索的索引速度。如果你想分配 `NULL` 到一個可選的字符串屬性，你會收到一個 `ConstraintError` 異常。你可以通過設置 `nullable=True` 屬性改變這種行為。然後空字符串和 `NULL` 在空字符串屬性列都將是允許的，但是很少需要這麽做。

Oracle 數據庫中，空字符串和`NULL`是同樣處理的。因此所有的 Oracle 數據庫中的 `Optional` 屬性， `nullable` 將會自動設置為 `True`。

如果一個字符串屬性被作為唯一鍵或者作為唯一鍵組合的一部分，它的 `nullable` 將會自動設置為 `True`。

### 主鍵
`PrimaryKey` 定義了數據庫表中的主鍵。每一個實體都因該有一個主鍵。如果主鍵沒有明確的指定，Pony 會隱含的創建。讓我們看以下例子：
```python
class Product(db.Entity):
    name = Required(str, unique=True)
    price = Required(Decimal)
    description = Optional(str)
```
這個實體的定義方式和下面的方式是等價的：
```python
class Product(db.Entity):
    id = PrimaryKey(int, auto=True)
    name = Required(str, unique=True)
    price = Required(Decimal)
    description = Optional(str)
```
Pony 自動添加的的主鍵總是名字叫 `id` 類型是 `int`。`auto=True` 參數意味著這個屬性將會使用數據的自增或者序列功能自動增加。

如果你自己指定了主鍵，那麽類型和名字都是自定義的。例如我們可以指定 `Customer` 實例對象使用 custormer's 的 email 作為主鍵。
```python
class Customer(db.Entity):
   email = PrimaryKey(str)
   name = Required(str)
```
### Set
一個 `Set` 屬性聲明了一個集合。你可以指定另外一個實體對象作為 `Set` 屬性的類型。這就是定義“對多”關系的方法，可以是“多對多”或者“一對多”。目前，Pony 還不能用 `Set`使用原始類型。我們計劃以後加入這個特色功能。我們將在《使用關系》章節中更詳細的討論。

### Composite keys
Pony 完全支持 Composite keys，為了定義一組唯一鍵，你需要用 `Required` 定義每個鍵，然後將它們組合成組合主鍵。
```python
class Example(db.Entity):
    a = Required(int)
    b = Required(str)
    PrimaryKey(a, b)
```
這裏 `PrimaryKey(a, b)` 並沒有創建新的屬性，而是將括號中指定的鍵值組合為組合主鍵。每個實體對象，有且只有一個組合主鍵。
為了創建第二個組合鍵，你需要像之前一樣聲明他們，然後使用 `composite_key`組合他們。
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_key(a, b)
```
在數據庫中，`composite_key(a,b)` 將會對應為 `UNIQUE("a","b")`約束條件。
如果只有一個屬性需要被設置為唯一的，你可以在創建鍵的地方指定 `unique=True` 屬性。
```python
class Product(db.Entity):
    name = Required(str, unique=True)
```
### Composite indexes
使用 `composite_index()` 就可以直接創建組合索引以加速數據檢索，它可以用來組合兩個或多個屬性：
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_index(a, b)
```
既可以使用屬性，也可以使用屬性的名字。
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_index(a, 'b')
```
如果你想對一個字段創建非唯一性的索引，你可以給這個屬性指定 `index` 屬性。這個屬性將在本章之後的章節詳細描述。

## 屬性的數據類型
實體對象的屬性就像的類型的一個屬性那樣聲明，使用語句 `sttr_name = kind(type)`。定義屬性的時候能用的數據類型如下：

- str
- unicode
- int
- float
- Decimal
- datetime
- date
- time
- timedelta
- bool
- buffer - used for binary data in Python 2 and 3
- bytes - used for binary data in Python 3
- LongStr - used for large strings
- LongUnicode - used for large strings
- UUID

`buffer` 和 `bytes` 類型是以二進制形式（BLOB）存儲在數據庫中的。`LongStr` 和 `LongUnicode` 是以 CLOB 形式存儲。

從 Pony Release 0.6 開始，增加了 Python3 的支持，我們建議使用  `str` 和 `LongStr` 替代 `Unicode` 和 `LongUnicode`。

我們知道，Python3 和 Python2 處理字符串的方法是不同的。Python2 提供兩種字符串類型 —— `str`(byte string) 和 `unicode` (unicode string)，因為 Python3 中用 `str` 代替了 unicode string ，移除 `unicode`。

在 release 0.6 之前，Pony 將 `str` 和 `unicode` 在數據庫中都存儲為 unicode 。但是對於 `str`的屬性，在讀取的時候必須將 unicode 轉為 byte string。從 Pony Release 0.6 開始，Python2 中的 `str` 類型和 `unicode` 類型具有相同的行為。`str` 和 `unicode` 定義的屬性完全一樣 —— 在 Python 中和數據庫中都是 unicode string。`LongStr` 和 `LongUnicode` 也是這樣。。`LongStr` 現在只是 `LongUnicode` 的別稱。如下定義，在 Python 中和數據庫中都是 unicode。
```python
attr1 = Required(str)
# is the same as
attr2 = Required(unicode)

attr3 = Required(LongStr)
# is the same as
attr4 = Required(LongUnicode)
``` 
Python2 裏，如果你需要表示字節數組，可以使用 `buffer` 類型，Python3 裏，已經移除了 `buffer` 類型，可使 `bytes` 達到相同的目的。但是為了向後兼容性，我們依然保留了 `buffer` 作為 `bytes` 的別稱。如果你使用 `import * from pony.orm`，別稱也就已經導入了。

如果你的程序需要兼容 Python2 和 Python3 ，你應該使用 `buffer` 類型來表示二進制的屬性。如果你的程序值運行在 Python3 下，你可以使用 `bytes`。
```python
attr1 = Required(buffer) # Python 2 and 3

attr2 = Required(bytes) # Python 3 only
```
如果 Python2 中能使用 `bytes` 作為 `buffer`的別稱那就`bytes` 作為最好了。但是那是不可能的，因為 [Python2.6 中增加了`bytes` 作為 `str` 的同義詞](https://docs.python.org/2/whatsnew/2.6.html#pep-3112-byte-literals)。

為了定義兩個類型之間的關系，你也可以使用另外一個實體類型作為屬性的類型。

## 屬性的選項

基於變量的位置或關鍵詞，你可以為屬性定義額外的選項。

### 字符串最大長度
`String` 類型接收一個基於變量位置的參數作為字段最大長度的限定。
```python
name = Required(str, 40)   #  VARCHAR(40)
```
### 整數最大值
對於 `int` 類型，你可以使用`size`關鍵詞指定數據庫中使用的整數類型的大小。這個參數接收一個數作為數據庫中整數占用的位數。允許的值是 8,16,24,32和64。
```python
attr1 = Required(int, size=8)   # 8 bit - TINYINT in MySQL
attr2 = Required(int, size=16)  # 16 bit - SMALLINT in MySQL
attr3 = Required(int, size=24)  # 24 bit - MEDIUMINT in MySQL
attr4 = Required(int, size=32)  # 32 bit - INTEGER in MySQL
attr5 = Required(int, size=64)  # 64 bit - BIGINT in MySQL
```
你可以使用 `unsigned` 參數指定這個屬性是無符號的。
```python
attr1 = Required(int, size=8, unsigned=True) # TINYINT UNSIGNED in MySQL
```

`unsigned` 參數默認是 `False` 如果 `unsigned` 是 `True`，但是沒有指定 `size` 參數，`size` 將默認為 32位。

如果當前數據庫不支持指定的屬性大小，則使用下一個更大的屬性。例如，PostgreSQL 沒有 `MEDIUMINT` 類型，如果指定 24位，那麽使用 `INTEGER` 類型。

只有 MySQL 完全支持無符號類型。對於其他數據庫將使用能夠包含指定的無符號類型的所有值的有符號類型的類型。例如，在 PostgreSQL 中，16位無符號的屬性，將會使用 `INTEGER` 類型。 64位的無符號數只能在 MySQL 和 Oracle 中實現。

當數據的大小指定了， Pony 會自動分配這個屬性的 `min` 和 `max` 值。例如，一個有符號的 8位屬性，可以接收 `min` 為 -128 和 `max` 為 127，8位無符號屬性，可以接收 `min` 為 0，`max` 為 255。如果有需要你可以覆蓋默認的 `min` 和 `max` 的值，但是不能超過位數指定的範圍。

>註
從 Pony Release 0.6 開始，不建議使用 `long` 類型，如果你要存儲 64位的整數，需要使用 `int` 類型和 `size=64` 作為替代。如果沒有指定 `size` 參數，Pony 會使用數據庫的默認整型。

### 小數的精度和進制
對於 `Decimal` 類型， 你可以指定精度和進制。
```python
price = Required(Decimal, 10, 2)   #  DECIMAL(10, 2)
```

### 日期和時間的精度

`datetime` 和 `time` 類型接收基於變量位置的精度參數。對於大多數數據庫為 6。

對於 MySQL 數據庫，默認值是 0。MySQL 5.6.4 之前，`DATETIME` 和 `TIME` 字段，[無法存儲分秒](http://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)。MySQL 5.6.4 之後，如果你指定了精度參數為 6，就可以使用分秒。
`dt = Required(datetime, 6)`

### 其他屬性選項

其他能被設為屬性選項的參數：

- unique 
布爾值，決定屬性是否唯一

- auto
布爾值，只能主鍵屬性使用。如果 `auto=True`，那麽屬性的值將會自增，基於數據庫提供的計數器或者序列。

- default
允許你給屬性指定默認值

- sql_default
這個屬性允許你指定默認的 SQL 語句，這將在 `CREATE TABLE SQL` 命令執行時起作用。例如：
`created_at = Required(datetime, sql_default='CURRENT_TIMESTAMP')`
當你有一個 `Required` 屬性，而且它的值能在 `INSERT` 命令執行時計算出來，那麽使用 `sql_default=True` 會很方便（例如，觸發器）。默認是 `None`。

- index
允許你控制字段的索引生成。`index=True` —— 將會使用默認的名字創建索引。 `index='index_name'` —— 使用指定的名字創建索引。`index=False` —— 跳過索引的生成。如果沒有指定 `index`選項，Pony 依然會使用默認名字創建外鍵的索引。

- sql_type
如果你想對字段創建指定的 SQL 類型，使用這個選項。

- lazy
布爾值，載入對象時延緩載入屬性。設置為 `True`，意味著當你直接調用這個屬性之前，這個屬性是不會載入的。`LongStr` 和 `LongUnicode` 的 `lazy` 默認設置為 `True`，其他類型默認設置為 `False`。

- cascade_delete
布爾值，控制是否刪除關聯的對象。`True` 意為 Pony 總是級聯的刪除，即使關系的另外一邊定義是 `Optional`， `False`意為 Pony 總是不做級聯的刪除。如果關系的另外一邊定義是 `Required`，而且 `cascade_delete=False`，那麽嘗試刪除時會拋出一個異常 `ConstraintError`。

- column
指定數據庫表中對應字段的名字。

- columns
制定一個列表，用作數據庫表中對應字段組合的名字

- reverse
指定關系中另外一邊，對應的屬性。

- reverse_column
用於多對多關系，指定中間表中對應字段的名字。

- reverse_columns
用於多對多關系，如果實體的主鍵是組合鍵時，指定中間表中對應字段的名字。

- table
多對多關系中指定中間表的名字。

- nullable
布爾值，`True` 意為數據庫中該字段可以設置為 `NULL`，大多數時候你不需要指定這個參數，因為 Pony 會設置核實的值。

- volatile
通常是你指定了 Python 屬性的值，Pony 存儲這個值到數據庫。但是有時候在數據庫中有一些邏輯，可能修改一個字段的值。例如，你可以在數據庫中有一個觸發器，會修改最後訪問的時間戳。這種情況下你可能需要讓 Pony 忘掉對象的屬性更新時發送到數據庫中的值，在下次訪問時使用數據庫中的值。設置 `volatile=True` 可以讓 Pony 知道這個屬性的值可能在數據庫中被修改。
`volatile=True` 可以和 `sql_default=True` 一起使用，如果這個屬性完全是由數據庫創建和更新的。
如果當前事務正在使用的值被別的事務修改了，你會得到一個異常：`UnrepeatableReadError: Value ... was updated outside of current transaction`。Pony 之所以要明確提示是因為這種情況可能會破壞應用的業務邏輯，如果你不需要這種競爭性保護，你可以設置 `volatile=True`。

- sequence_name
對於 Oracle 數據庫，指定 `PrimaryKey` 屬性使用的序列的名字。

- py_check
這個屬性允許你制定一個用來檢查賦給屬性的值的函數。這個函數必須返回 `True` 或者`False`。也可以在檢查出錯時拋出 `ValueError` 異常。

- min
指定數字型屬性（int, float, Decimal）的最小值，如果你賦值的值小於這個值，會拋出 `ValueError` 異常。

- max
指定數字型屬性（int, float, Decimal）的最大值，如果你賦值的值大於這個值，會拋出 `ValueError` 異常。

## 實體類型的繼承
Pony 中的實體對象的繼承和 Python 中規則類似。讓我們考慮一個例子，實體對象 `Student` 和 `Professor` 繼承自 `Person` ：

```python
class Person(db.Entity):
    name = Required(str)

class Student(Person):
    gpa = Optional(Decimal)
    mentor = Optional("Professor")

class Professor(Person):
    degree = Required(str)
    students = Set("Student")
```

子類繼承了基類 `Person` 的所有屬性和關系。有些映射器（例如，Django）存在一個問題，當使用基類進行查詢時沒有返回正確的類型：對於指定的實例，查詢結果只包含實例的繼承來的部分。Pony 沒有這個問題，你總是可以得到正確的實體對象的實例：
```python
for p in Person.select():
    if isinstance(p, Professor):
        print p.name, p.degree
    elif isinstance(p, Student):
        print p.name, p.gpa
    else:  # somebody else
        print p.name
```
為了得到正確的實體對象實例，Pony 使用了額外的區分字段。默認情況下這是一個字符串字段，Pony 使用它來存儲實體對象類型的名字。
`classtype = Discriminator(str)`
默認情況下，對於繼承的實體類型，Pony 隱式的的創建了 `classtype` 屬性。你可以自定義區分字段的名字和類型。如果你改變了區分字段的類型，那就必須指定每一個實體對象的 `_discrimintator_`。讓我們看下以下的例子，使用 `cls_id` 作為區分字段，使用 `int` 類型。
```python
class Person(db.Entity):
    cls_id = Discriminator(int)
    _discriminator_ = 1
    ...

class Student(Person):
    _discriminator_ = 2
    ...

class Professor(Person):
    _discriminator_ = 3
    ...
```

## 多重繼承
Pony 也支持多重繼承。如果要使用多重繼承，那麽當前類的所有父類都必須繼承自同一個基類。（菱形繼承）。
讓我們看一下這個學生可以當助教的例子。我們引入 `Teacher` 類型，並從它衍生出 `Professor` 和 `TeachingAssistant`。其中， `TeachingAssistant` 繼承自 `Student` 和 `Teacher` 兩個基類。

```python
class Person(db.Entity):
    name = Required(str)

class Student(Person):
    ...

class Teacher(Person):
    ...

class Professor(Teacher):
    ...

class TeachingAssistant(Student, Teacher):
    ...
```
`TeachingAssistant` 的對象，既是 `Student` 也是 `Teacher` 的實例，繼承了它們所有的屬性。多重繼承在這裏是可行的是因為 `Student` 和 `Teacher` 有相同的基類 `Person`。
繼承是非常強大的，但是要明智的使用它，對於非常簡單的數據表，繼承的用處不大。

## 繼承在數據庫中的應用
在數據庫中有三種方法實現繼承：
1. 單一表繼承法：繼承樹中的的所有子類都映射到一個表。
2. 分類表繼承法：繼承樹中的的每層子類都應到到一個單獨的表，但是每個表只存儲不是從父類繼承的部分。
3. 完整表繼承法：繼承中的的每層子類都應到到一個單獨的表，每個表都存儲了存儲實體對象的屬性和它的所有的父類。

第三種途徑的問題是沒有單獨的一個表可以存儲主鍵，那就是為什麽這種方法很少使用。

第二種方法是最常用的，這就 Django 中繼承實現所使用的。這種方法的缺點是取數據的時候需要並表，可能導致性能下降。

Pony 使用第一種方式，所有繼承樹中的實體對象到映射到一張單獨的表。這是最有效率的方法，因為不需要並表查詢。這種方法也有它的缺點：

- 數據表中的每一條數據可能都有一些字段不能用，因為他們屬於其他的實體類型。這不是一個大問題，因為空的字段都會被置位 `NULL`，不占太大空間。
- 如果繼樹中有很多的實體對象，數據表可能會有非常多的字段。不同的數據庫對最大字段數的限制是不一樣的，但是一本都非常大。

第二種方法還有一個好處：當需要增加一個新的實體對象的時候，基類的表是不需要變動的。Pony 以後將會這種方式。

## 自定義映射
當 Pony 從實體對象創建數據表的時候，默認使用實體對象的名字作為表名，使用屬性的名字作為字段名，但是你可以重寫這個行為。

數據表名字也不總是等於實體對象的名字： 在 MySQL 和 PostgreSQL，數據表名來自實體對象的名字轉換為小寫，在 Oracle 中轉為大寫。總是可以通過讀取 `_table_` 屬性來知道對應的數據表。

如果你需要設置你自己的數據庫表名，使用 `_table_`屬性。
```python
class Person(db.Entity):
    _table_ = "person_table"
    name = Required(str)
```
如果你需要設置你自己的字段名，使用 `column` 選項。
```python
class Person(db.Entity):
    _table_ = "person_table"
    name = Required(str, column="person_name")
```
對於組合屬性，使用 `columns` 和名字的列表：
```python
class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    lectures = Set("Lecture")
    PrimaryKey(name, semester)

class Lecture(db.Entity):
    date = Required(datetime)
    course = Required(Course, columns=["name_of_course", "semester"])
```
在這個例子中，我們重寫了組合屬性  `Lecture.course` 的字段名。默認情況 Pony 將使用 `"course_name"` 和 `"course_semester"`， 由實體對象名字和屬性名字連接而成方便程序員識別。

如果你想要自定義多對多關系中的中間表的名字，你需要使用對 `Set` 屬性使用 `column` 或 `columns` 選項，讓我們看下面的例子：

```python
from pony.orm import *

db = Database("sqlite", ":memory:")

class Student(db.Entity):
    name = Required(str)
    courses = Set("Course")

class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    students = Set(Student)
    PrimaryKey(name, semester)

sql_debug(True)
db.generate_mapping(create_tables=True)
```
默認情況下，為了存儲 `Student` 和 `Course` 的多對多關系, Pony 會創建 `"Course_Student"` 中間表（中間表的名字是實體對象的名字按照首字母字母表的順序相連）。這個表將會有三個字段 `"course_name"`, `"course_semester"` 和 `"student"` ，`Course` 的組合主鍵占用兩個，和 `Student` 的占用一個字段。如果我們要 `"Study_Plans"` 包含字段： `"course"`, `"semester"` 和 `"student_id"`，以下代碼片段實現了這個目的：

```python
class Student(db.Entity):
    name = Required(str)
    courses = Set("Course", table="Study_Plans", columns=["course", "semester"]))

class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    students = Set(Student, column="student_id")
    PrimaryKey(name, semester)
```
更多關於自定義映射的內容可以查看 [PonyORM 包中帶的例子](https://github.com/ponyorm/pony/blob/orm/pony/orm/examples/university.py)。 
