# 關系

實體對象可以和其他實體對象建立關系。一條關系是由關系兩邊的實體對象的兩條屬性定義的：
```python
class Customer(db.Entity):
    orders = Set("Order")

class Order(db.Entity):
    customer = Required(Customer)
```
在上面的示例中，我們有兩個關系屬性：`orders` 和 `customer`。當我們定義 `Customer` 的時候， `Orders` 還沒有定義，所以需要使用引號將 `Orders`o 括起來。也可以使用 lambda 表達式：

```python
class Customer(db.Entity):
    orders = Set(lambda: Order)
```
如果你想要 IDE 幫你檢查實體對象名字的拼寫高亮拼寫錯誤，這種方法很有用。

有一些映射器（例如，django）只需要在一邊定義關系就可以了。Pony 需要在關系的兩邊都明確的定義（就像 Python 之禪說的：明確比暗示好），用戶在每個角度都能看到所有的關系定義。

所有的關系都是雙向的。如果你更新了關系的一邊，另一邊會自動更新。如果我們創建了一個 `Orders` 的實例，customer 的 orders 集合會更新，添加了這條新的 order。

有三種類型的關系：一對一，一對多，多對多。一對一關系很少使用，大多數關系形式都是一對多和多對多。如果兩個實體對象擁有一對一關系，那基本上就意味著這兩個實體對象可以合並為一個。如果你的數據表中包含很多一對一關系，那就意味著你需要重新思考實體對象的定義了。

## 一對多關系
這裏有一個一對多關系的例子：
```python
class Order(db.Entity):
    items = Set("OrderItem")

class OrderItem(db.Entity):
    order = Required(Order)
```
上例中，沒有 order 的情況下就不能建立 `OrderItem` 的實例，如果我們想建立一個不需要指定 order 的 `OrderItem` 實例，我們可以指定 `order` 屬性為 `Optional`。

```python
class Order(db.Entity):
    items = Set("OrderItem")

class OrderItem(db.Entity):
    order = Optional(Order)
```

## 多對多關系
為了建立多對多關系，你需要在關系的兩邊都使用 `Set` 屬性。
```python
class Product(db.Entity):
    tags = Set("Tag")

class Tag(db.Entity):
    products = Set(Product)
```
為了在數據庫中實現這個關系，Pony 將會創建一個中間表。這是眾所周知的在數據庫中建立多對多關系的方法。

## 一對一關系
為了建立一對一關系，關系屬性需要設置為 `Optional-Required` 或 `Optional-Optional`:

```python
class Person(db.Entity):
    passport = Optional("Passport")

class Passport(db.Entity):
    person = Required("Person")
```
兩個屬性都是 `Required` 是不允許的，因為這不合理。

## 自引用
實體對象可以通過自引用建立指向自己的關系。這種關系可以有兩種類型：相稱的和不相稱的。不相稱的關系就是一個實體對象裏的兩個屬性建立的關系。

相稱的關系特殊的地方就是這條關系只有一條屬性，這條屬性定義了關系的兩邊。這種關系可以是一對一也可以是多對多。如下是自引用關系的例子：
```python
class Person(db.Entity):
    name = Required(str)
    spouse = Optional("Person", reverse="spouse") # symmetric one-to-one
    friends = Set("Person", reverse="friends")    # symmetric many-to-many
    manager = Optional("Person", reverse="employees") # one side of non-symmetric
    employees = Set("Person", reverse="manager") # another side of non-symmetric
```

## 兩個實體對象間的多種關系
當兩個實體類型之間的關系多於一條的時候，Pony 需要 reverse 屬性來加以區分。這是為了讓 Pony 知道那兩個屬性是一對。像我們考慮一下這個 user 既可以寫微博，又可以給微博點讚的數據表。
```python
class User(db.Entity):
    tweets = Set("Tweet", reverse="author")
    favorites = Set("Tweet", reverse="favorited")

class Tweet(db.Entity):
    author = Required(User, reverse="tweets")
    favorited = Set(User, reverse="favorites")
```
在上例中，我們必須指定 `reverse` 屬性。當你嘗試生成映射但是沒有提供`reverse` 屬性時，你會得到一個異常  `pony.orm.core.ERDiagramError: "Ambiguous reverse attribute for Tweet.author"`。之所以會這樣是因為 `author` 屬性在技術角度既可以和 `tweets` 配對，又可以和 `favorites` 配對，Pony 沒有足夠的信息區分。
