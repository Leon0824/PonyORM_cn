# 事務

事務是一組邏輯上的工作單元，可以由一個或多個語句組成。事務是原子的，那就是說，當通過事務改變數據庫的時候，要麽事務成功所有的改變都成功，要麽事務回滾，所有的改變都沒做。

Pony 使用數據庫 session 來實現事務關系。

## 通過數據庫 session 來工作
和數據庫交互的代碼都必須工作在數據庫 session 之下。session 定義了數據庫會話的邊界。由數據庫操作的每一個線程會建立一個單獨的數據庫 session ，使用一個單獨的 Identity Map。Identity Map 像個緩存一樣工作，當你通過主鍵或者唯一性鍵查詢對象的時候，如果身份映射表已經有了，就避免了再次查詢數據庫。為了使用數據庫 session 來操作數據庫，你可以使用裝飾器 `@db_session` 或者 `db_session` 上下文管理器。當一個 session 結束的時候，它完成了以下動作。

- 如果數據被改變了，而且沒有異常，那麽提交事務。如果有異常發生，回滾事務。
- 將數據庫連接歸還到數據庫連接池。
- 清空 Identity Map 緩存。

如果你在必要的地方沒有指定 `db_session`，Pony 會拋出異常 `TransactionError: db_session is required when working with the database`。

使用 `@db_session` 裝飾器的例子:
```python
@db_session
def check_user(username):
    return User.exists(username=username)
```

使用 `db_session` 上下文管理器的例子:
```python
def process_request():
    ...
    with db_session:
        u = User.get(username=username)
        ...
```

>註
當你使用 Python 交互器來操作時，你不用考慮數據庫會話的問題，因為 Pony 已經自動替你維護了。

當你嘗試訪問 `db_session` 區域外的實例的屬性（非數據庫自動創建的），你會得到一個異常 `DatabaseSessionIsOver`，例如：

```python
DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over
```

之所以有這種事情是因為，數據庫連接已經返還到連接池，事務已經關閉，已經不能傳送任何命令到數據庫。

當 Pony 從數據庫中讀取對象的時候，它將對象放到 Identity Map。之後，當你更新對象的屬性，創建珊處對象時，改變會首先增量的保存到 Identity Map。改變會在事務提交的時候，或者在調用 `select()`、 `get()`、 `exists()`、 `execute()` 之前，保存到數據庫。

### 事物的範圍
一般情況下，使用 `db_session`，你就有了一個事務。沒用明確的開啟事務的命令。事務將在第一條 SQL 語句發送到數據庫的時候執行。在發送第一條語句之前， Pony 會從連接池中獲取一個數據庫連接。之後的 SQL 語句都會使用相同的上下文執行。

>註
SQLite 的 Python 驅動不在 SELECT 語句上開啟事務。只有在能改變數據庫狀態的語句上才會開啟事務，有：INSERT, UPDATE, DELETE。其他驅動在任何命令上都會開啟事務，包括 SELECT 。

事務在使用 `commit()` 提交的時候結束，或者在 `rollback()` 調用的時候回滾，或者離開 `db_session` 區域。

```python
@db_session
def func():
    # a new transaction is started
    p = Product[123]
    p.price += 10
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```

Several transactions within the same db_session
### 一個 db_session 下的多個事務 

如果你需要在一個事務中包含多個數據庫 session，你可以 session 範圍下任何時候調用 `commit()` 或 `rollback()`，同時下一條語句會開啟一個新的事務，手動調用 `commit()` 之後, Identity Map 保留了數據緩存，你可以使用 `rollback()` 清除緩存。

```python
@db_session
def func1():
    p1 = Product[123]
    p1.price += 10
    commit()          # the first transaction is committed
    p2 = Product[456] # a new transaction is started
    p2.price -= 10
```

### db_session 的嵌套

如果你遞歸的進入 `db_session` 的區域，`@db_session`修飾的函數調用了另一個 `@db_session`修飾的函數，Pony 將不會創建新的 session，而是將原來的 session 共享給兩個函數。數據庫 session 將在離開最外層的 `db_session` 修飾器或這上下文管理器時關閉。

### db_session 的緩存

為了提高性能，Pony 在如下幾種情況緩存數據：

- 生成器表達式的執行結果。如果一條生成器表達式被調用了多次，只會向數據庫發送一次數據。這個緩存是實體對象程序全局的，不屬於單個的 session 。
- 數據庫創建或載入的對象。Pony 保留這些對象到 Identity Map ，離開 `db_session` 區域或者事務回滾的時候清除。
- 執行結果。相同參數的相同語句會直接從緩存中返回結果。實例改變一次緩存清空一次。離開 `db_session` 區域或者事務回滾的時候緩存清除。

### 多數據庫

Pony 可以同時使用多個數據庫。下面的例子中我們使用 PostgreSQL 存儲用戶信息，用 MySQL 存儲地址信息。
```python
db1 = Database("postgres", ...)

class User(db1.Entity):
    ...

db2 = Database("mysql", ...)

class Address(db2.Entity):
    ...

@db_session
def do_something(user_id, address_id):
    u = User[user_id]
    a = Address[address_id]
    ...
```
當退出 `do_something()` 函數時， Pony 對所有的數據庫執行 `commit()` 或 `rollback()`，如果有必要。

## 事務的相關函數

- commit()
使用 flush() 函數存儲當前 `db_session`下的所有修改，提交事務到數據庫。頂層的 `commit()` 會調用 當前事務所用到的數據庫對象的 commit() 方法。

- rollback()
回滾當前事務。頂層的 `rollback()` 會調用 當前事務所用到的數據庫對象的 rollback() 方法。

- flush()
將 `db_session` 緩存中的變動保存到數據庫，不包含提交數據變動。大多數情況，Pony 自動從數據庫會話緩存中將數據保存到數據庫，不需要你自己調用這個函數。有一種情況你需要調用它，當你想在 commit 之前獲取一個新對象自動獲取的主鍵的時候。

Pony 總是在執行
`select()`、 `get()`、 `exists()`、 `execute()` 和 `commit()` 前自動增量式保存變化到 `db_session` 緩存。
`flush()` 函數讓 `db_session` 緩存中的更新在當前事務下的數據庫訪問中生效。同時，`flush()`並沒有真正將數據存入數據庫。
頂層的 `flush()` 會調用 當前事務所用到的數據庫對象的 flush() 方法。

## db_session 的參數
之前已經提到 `db_session` 可以作為修飾器或者上下文管理器。`db_session` 可以接收以下參數。

- retry
接收一個整數值，指定嘗試提交這個事務的次數。這個參數只能在修飾器形式下使用。被修飾的函數不能直接調用 `commit()` 和 `rollback()`。當指定了這個參數，Pony 將緩存 `TransactionError` 以及它的派生類的異常，然後重置事務。默認情況下 Pony 只緩存 `TransactionError` 異常，但是這個列表可以被 `retry_exceptions`參數重寫。

- retry_exceptions
接收一個列表，指定這些異常將會導致事務重啟。默認情況下，這個參數值是 `[TransactionError]`。另外的選擇是指定一個回調函數，這個回調函數接收一個參數 —— 當前發生的異常。如果這個函數返回 `True`，事務將會重啟。

- allowed_exceptions
這個參數接收一個異常列表，當這些異常發生時，失誤不會回滾。例如，一些 HTTP 框架通過異常來觸發跳轉。

- immediate
接受一個布爾值，默認值是 `False`。一些數據庫（例如，SQLite，Postgres）只有當提交更改數據的語句（UPDATE，INSERT，DELETE）時開啟一個事務，SELECT 命令則不會。如果你想在 SELECT 時也開啟事務，可以通過傳遞 `True` 到這個參數。通常情況下沒必要改變這個值。

- serializable
接受一個布爾值，默認值是 `False`。允許你設置 SERIALIZABLE 串行化的等級。

## 樂觀的並發控制
為了提升性能，Pony 默認使用樂觀的並發控制。基於這個觀點，Pony 並不獲取數據的鎖。而是確認並沒有別的會話嘗試讀取或修改相同的數據。如果檢測到沖突的修改，事務提交時拋出異常 `OptimisticCheckError, 'Object XYZ was updated outside of current transaction'`，然後回滾。

面對這種情況我們應該怎麽做？首先，這種行為在使用 [MVCC](http://en.wikipedia.org/wiki/Multiversion_concurrency_control) 模式的數據庫（例如，Postgre，Oracle）是很常見的。例如，在 Postgres 中，在不同事務同時修改相同的數據時，你會得到如下錯誤：
`ERROR: could not serialize access due to concurrent update`
當前事務被回滾，但是它可以被重新開始。為了自動重試事務，你可以在 `db_session` 修飾器使用 `retry` 參數（在本章後面查看更多細節）。

Pony 怎麽做這種樂觀的檢查？Pony 跟蹤每個對象的屬性的訪問。當用戶的代碼讀或修改一個對象的屬性時，Pony 檢查這個屬性的值是否有待提交到數據庫的殘余內容。這種方式可以避免丟失數據更新，有一種情況，當當前事務和並發的別的事務修改相同的對象時，當前事務覆蓋了數據了，掩蓋了之前的修改。

在樂觀的檢查中，Pony 只檢查用戶讀或寫的屬性。在 Pony 更新對象的時候，也是只更新用戶修改了的屬性。在這種運行方式下，兩個不同的事務更新對象的不同屬性，然後都成功了，這是可能的。

通常樂觀的並發控制可以提升性能，因為事務執行可以避免請求鎖和等待別的事務釋放鎖。這種方法在沖突很少和讀筆寫多得多的情況下表現良好。

## 悲觀的鎖

有些時候我們需要鎖定數據庫中的對象，來避免其他事務修改相同的記錄。在數據庫中可以使用 `SELECT FOR UPDATE` 語句。在 Pony 中要生成這種可以使用 `for_update` 方法。
 `select(p for p in Product if p.price > 100).for_update()`
上面語句選定的符合價格大於 100 的 Product 的實例都將被鎖定。在當前事務提交或回滾之後，鎖會被釋放。

如果你需要鎖定一個對象，你可以使用 `get_for_update` 方法。
`Product.get_for_update(id=123)`
當你嘗試用 `for_update` 鎖定一個對象的時候，如果它已經被另外的事務鎖定了，你的請求將不得不等待行級別的鎖被釋放。為了避免這種情況，使用 `nowait=True`:
```python
select(p for p in Product if p.price > 100).for_update(nowait=True)

Product.get_for_update(id=123, nowait=True)
```
這種情況下，如果選定的行不能被馬上鎖定，當前請求將會報告一個錯誤而不是等待。
消極的鎖的主要問題是性能下降，因為數據庫鎖的消耗和對並發的限制。

## 事務隔離和數據庫差別
隔離是一個屬性，定義了當一個事務修改了數據之後，多久別的並行事務能夠看到修改。ANSI SQL 標準規定了四個等級。

- READ UNCOMMITTED - 最不安全的等級
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE - 最安全的等級

當使用 SERIALIZABLE 等級的時候，每個事務都將數據庫看成事務開始時的一個快照。這個等級提供最大程度的數據隔離，但是比其他等級需要等多的資源。

這就是為什麽大多數數據庫使用更低的隔離等級以允許更好的並發。默認情況下，Oracle 和 PostgreSQL 使用 READ COMMITTED, MySQL - REPEATABLE READ. SQLite 只支持 SERIALIZABLE，但是 Pony 模擬了 READ COMMITTED，支持更好的並發。

如果你希望 Pony 使用 `SERIALIZABLE` 等級的事務，你可以給 `@db_session` 修飾器或 `db_session` 上文管理器指定 `serializable=True` 參數。

## READ COMMITTED vs. SERIALIZABLE 模式
在 `SERIALIZABLE` 模式，你總是需要應對得到 `“Can’t serialize access due to concurrent update”` 錯誤，而且還要重試事務直到成功。在 SERIALIZABLE 模式事務寫數據庫，總是需要在你的應用中寫重試循環。

在 `READ COMMITTED` 模式，如果你希望在並發的事務中避免新修改相同的數據，你應該使用 `SELECT FOR UPDATE`。但是這種情況下有可能導致數據庫[死鎖](http://en.wikipedia.org/wiki/Deadlock) —— 這種情況就是一個事務在等待另一個事務鎖定的資源。如果你的事務陷入死鎖，就需要重啟事務。所以你無論如何都需要一個重試循環。Pony 可以自動重試一個事務，如果你指定了 `retry` 參數給 `@db_session` 修飾器（ 不能是 `db_session` 上文管理器）。

```python
@db_session(retry=3)
def your_function():
    ...
```

### PostgreSQL
PostgreSQL 默認使用 READ COMMITTED 隔離等級。PostgreSQL 也提供自動提交模式。在這種模式下，每條 SQL 語句都運行在一個獨立的事務中。當你的應用只是讀取數據庫中的數據，自動提交模式可以更有效，因為沒必要開啟和關閉事務。數據庫直接提供了這個功能。從隔離這個角度來說，自動提交模式和 READ COMMITTED 隔離等級沒有什麽不同，在兩種情況下，你的應用都是馬上看到已經提交的數據。

Pony 自動在自動提交模式和明確的開啟一個事務之間切換，當你的應用需要使用 INSERT、 UPDATE 或 DELETE SQL 來原子的修改數據。

### SQLite
當使用 SQLite 的時候，Pony 的行為和使用 PostgreSQL 時類似：當事務啟動之後，讀取數據將會在自動提交模式執行。這種模式的隔離等級相當於 READ COMMITTED。這種方式下，並發的事務可以被同時執行而沒有死鎖的風險（Pony 不會拋出 `sqlite3.OperationalError: database is locked` 異常）。當你的代碼發布了非獨占聲明，Pony 會開啟一個事務，然後之後的 SQL 語句也會使用這個事務執行。事務將會是 SERIALIZABLE 隔離等級。

### MySQL
MySQL 默認使用 REPEATABLE READ 隔離等級。Pony 將不會在 MySQL 使用自動提交模式，因為這無利可圖。事務在第一條 SQL 語句發送到數據庫的時候開啟，即使這條語句是 SELECT 。

### Oracle
Oracle 默認使用 READ COMMITTED 隔離等級。Oracle 沒有自動提交模式。事務在第一條 SQL 語句發送到數據庫的時候開啟，即使這條語句是 SELECT 。

## Pony 怎樣避免丟失更新數據
低的隔離等級在許多用戶同時訪問數據的時候提升了性能，但是這也有可能導致數據庫異常，例如丟失更新。

讓我們考慮一種情境。我們有兩個賬戶，我們需要提供一個函數將一個賬戶的錢轉到另一個賬戶。在轉賬的過程中，我們需要檢查賬戶中的錢是否足夠。

假如我們使用 Django ORM 。下面是一個可能的例子實現了這個函數。

```python
@transaction.atomic
def transfer_money(account_id1, account_id2, amount):
    account1 = Account.objects.get(pk=account_id1)
    account2 = Account.objects.get(pk=account_id2)
    if amount > account1.amount:    # validation
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account1.save()
    account2.amount += amount
    account2.save()
```
在 Django 中默認，`save()` 在單獨的一個事務中運行。如果在第一個 `save()` 之後有一個錯誤，這筆錢就消失了。即使沒有錯誤，如果另外一個事務在兩個 `save()` 之間訪問賬戶，結果將是錯誤的。為了避免這個問題，所有的操作都必須在一個事務中完成。我們可以通過用 `@transaction.atomic` 裝飾器裝飾這個函數。

但是否即使這樣，我們還會碰到一個問題。如果兩個支行同時向第三個賬戶匯款，操作都將會被執行。每個函數都傳遞了值但是後完成的事務將會覆蓋先完成的那個。這種異常叫做“丟失更新數據”。

有三種方法來避免這種異常:

- 使用 SERIALIZABLE 隔離等級
- SELECT FOR UPDATE 替代 SELECT
- 使用樂觀的檢查

如果你使用 SERIALIZABLE 隔離等級，數據庫將不會允許第二個請求，在提交階段會拋出一個異常。這種方法的缺點是需要更多的系統資源。

如果你使用 SELECT FOR UPDATE ，事務將會競爭數據庫，先到的會鎖定這行數據，另外一個將會等待。

樂觀的檢查會不多占用系統資源，也不會鎖定數據庫。它通過確保數據不會在從數據庫讀取到提交的期間被改變來排除丟失數據異常。

Django 中避免丟失數據的唯一方法是使用 SELECT FOR UPDATE，而且你必須明確的指定它。如果你遺忘了或者沒有意識到，那麽這個問題就是就會存在於你的業務邏輯中，有可能導致數據遺失。

三種方法 Pony 都允許使用，第三種方法，樂觀的檢查，默認是開啟的。使用這種方式，Pony 完全避免了數據更新時的丟失問題。使用樂觀的檢查也允許更高的並發，因為它並不鎖定數據庫，也不需要額外的資源。

在 Pony 中類似的轉賬函數將會是這個樣子：

```python
@db_session(serializable=True)
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

使用 SELECT FOR UPDATE 的方法
```python
@db_session
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

使用樂觀檢查的方法
```python
@db_session
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

最後這種方法是 Pony 默認的，而且不需要明確的增加別的東西。
