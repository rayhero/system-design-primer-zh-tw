# 設計 Mint.com

*注意：本文件某些連結直接連到 [系統設計主題的索引](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E7%B3%BB%E7%B5%B1%E8%A8%AD%E8%A8%88%E4%B8%BB%E9%A1%8C%E7%9A%84%E7%B4%A2%E5%BC%95)來避免 重複的內容。你可以參考連結來取得相關重點、設計的取捨和選擇。*

## 步驟一：描述使用情境與限制

> 蒐集問題的需求、資訊和範圍。
> 透過詢問問題來瞭解使用情境和限制。
> 討論你的假設。

在這裡沒有面試者會幫你釐清上面的問題，所以我們會預先定義一些使用情境和限制。

### 使用情境

#### 我們將所要解決的問題限縮在以下範圍

* **使用者** 連接到一個財務帳戶
* **服務內容** 從帳戶中取得交易紀錄
    * 每日更新
    * 進行交易的分類
        * 允許使用者手動自己手動覆蓋
        * 沒有自動重新分類功能
    * 每月針對各個類別進行花費分析
* **服務內容** 建議的預算
    * 允許使用者設定預算上限
    * 當接近或超過預算上限時會通知使用者
* **服務** 是高可用的

#### 不包含在此範圍

* **服務** 提供額外的日誌記錄與分析功能

### 限制與假設

#### 狀態假設

* 流量不是均勻分布的
* 每日的自動更新帳戶功能僅適用於過去三十天內處於活動狀態的使用者
* 新增或刪除一個帳戶的功能相對較少使用
* 預算通知不需要是即時的
* 1000 萬個使用者
    * 每個 user 有 10 個預算的類別 = 總共有 1 億筆的預算資料
    * 幾個類別範例：
        * 租屋 = $1,000
        * 食物 = $200
        * 加油 = $100
    *販賣者被用來決定每個交易的類別
        * 50,000 個販賣者
* 3000 萬個帳戶
* 每月 50 億筆交易資料
* 每月 5 億次的讀取請求
* 寫入和讀取的比例為 10:1
    * 寫入的需求很大，因為使用者每天都會進行交易，但相對比較少的人會每天訪問網站

#### 計算使用量

**向你的面試人員詢問你是否可以用比較粗略的方式來計算使用量**

* 每筆資料的大小：
    * `user_id` - 8 bytes
    * `created_at` - 5 bytes
    * `seller` - 32 bytes
    * `amount` - 5 bytes
    * 總共約： ~50 bytes
* 每月 250 GB 新的交易資料量
    * 每筆交易 50 bytes * 每月 50 億筆交易資料
    * 3 年內有 9 TB 的新交易資料
    * 假設絕大多數都是新的交易資料，而不是更新既有的資料
* 平均每秒 2000 筆交易
* 平均每秒 200 個讀取的請求

一些筆記：

* 每月 250 萬秒
* 每秒 1 次請求 = 每月 250 萬次請求
* 每秒 40 次請求 = 每月 1 億次請求
* 每秒 400 次請求 = 每月 10 億 次請求

## 步驟二：進行高階設計

> 提出所有重要元件的高階設計

![Imgur](http://i.imgur.com/E8klrBh.png)

## 步驟三：設計核心元件

> 深入每個核心元件的細節

### 使用案例：使用者連接到一個財務帳戶

我們可以在 [關連式資料庫](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E9%97%9C%E9%80%A3%E5%BC%8F%E8%B3%87%E6%96%99%E5%BA%AB%E7%AE%A1%E7%90%86%E7%B3%BB%E7%B5%B1rdbms) 中為 1000 萬名使用者儲存相關資料。我們應該要進行 [SQL 或 NoSQL 的相關使用案例](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#sql-%E6%88%96-nosql) 討論。

* **客戶端** 發出一個請求到 **網頁伺服器** 上，這個伺服器是一個 [反向代理伺服器](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E7%B6%B2%E9%A0%81%E4%BC%BA%E6%9C%8D%E5%99%A8)。
* **網頁伺服器**會轉送請求到**帳戶 API 伺服器**。
* **帳戶 API 伺服器**更新 **SQL 資料庫**的 `accounts` 資料表中的相關資訊。

**向你的面試者詢問他預期你的程式碼要寫到什麼程度**.

`accounts` 資料表包含以下結構：

```
id int NOT NULL AUTO_INCREMENT
created_at datetime NOT NULL
last_update datetime NOT NULL
account_url varchar(255) NOT NULL
account_login varchar(32) NOT NULL
account_password_hash char(64) NOT NULL
user_id int NOT NULL
PRIMARY KEY(id)
FOREIGN KEY(user_id) REFERENCES users(id)
```

我們會在 `id`、`user_id` 和 `created_at` 三個欄位建立 [索引](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E4%BD%BF%E7%94%A8%E6%AD%A3%E7%A2%BA%E7%9A%84%E7%B4%A2%E5%BC%95) 來增加讀取速度 (log-time 的讀取時間，而不是掃描整張資料表)，同時確保資料會被保存在記憶體中。從記憶體中循序讀取 1MB 的資料大約需要 250 毫秒，但相同的資料，從 SSD 大概需要四倍以上的時間，而一般的硬碟則需要八十倍以上的時間<sup><a href=https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E6%AF%8F%E5%80%8B%E9%96%8B%E7%99%BC%E8%80%85%E9%83%BD%E6%87%89%E8%A9%B2%E7%9F%A5%E9%81%93%E7%9A%84%E5%BB%B6%E9%81%B2%E6%95%B8%E9%87%8F%E7%B4%9A>1</a></sup>。

我們會使用公開的 [**REST API**](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E5%85%B7%E8%B1%A1%E7%8B%80%E6%85%8B%E8%BD%89%E7%A7%BB-rest)：

```
$ curl -X POST --data '{ "user_id": "foo", "account_url": "bar", \
    "account_login": "baz", "account_password": "qux" }' \
    https://mint.com/api/v1/account
```

內部通訊上，我們可以使用 [遠端程式呼叫](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E9%81%A0%E7%AB%AF%E7%A8%8B%E5%BC%8F%E5%91%BC%E5%8F%AB-rpc) 的方式。

下一步，我們的服務會從帳戶中取出交易資訊。

### 使用案例：服務從帳戶中取出交易資訊

We'll want to extract information from an account in these cases:

* The user first links the account
* The user manually refreshes the account
* Automatically each day for users who have been active in the past 30 days

我們希望從帳戶中取出以下使用情境的資訊：

* 使用者第一次連接到帳戶
* 使用者手動更新帳戶內容
* 對於過去 30 天內持續活躍的使用者，每天自動的取得資訊

資料流：

* **客戶端**發送請求到**網頁伺服器**。
* **網頁伺服器**轉發請求到**帳戶 API 伺服器**。
* **帳戶 API 伺服器**發送一個工作到**佇列**中，例如：Amazon 的 SQS 或 [RabbitMQ](https://www.rabbitmq.com/)。
    * 擷取交易資料可能會花一些時間，我們可能希望採用 [異步處理加上佇列](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E9%9D%9E%E5%90%8C%E6%AD%A5%E6%A9%9F%E5%88%B6) 的機制來進行，即使這可能會增加額外的複雜度。
* 這個**交易資訊擷取的服務**包含以下內容：
    * 從**佇列**中取出工作，並針對帳戶擷取交易資訊，將結果視為原始日誌檔案存到**物件資料庫**中。
    * 使用**類別服務**來分類每個交易紀錄
    * 使用**預算服務**按照類別計算每月總支出
        * **預算服務**使用**通知機制**來讓使用者知道他們是否接近或超過所設定的預算
    * 使用分類好的交易資訊來更新**SQL 資料庫**中的 `transactions` 資料表
    * 使用按照類別匯總後的每月支出來更新**SQL 資料庫**中的 `monthly_spending` 資料表
    * 透過**通知服務**來通知使用者他的交易資料已經處理完畢：
        * 使用**佇列服務**來異步的發送通知

`transactions` 資料表可能包含以下的結構：

```
id int NOT NULL AUTO_INCREMENT
created_at datetime NOT NULL
seller varchar(32) NOT NULL
amount decimal NOT NULL
user_id int NOT NULL
PRIMARY KEY(id)
FOREIGN KEY(user_id) REFERENCES users(id)
```

We'll create an [index](https://github.com/donnemartin/system-design-primer#use-good-indices) on `id`, `user_id `, and `created_at`.

我們可以在 `id`、`user_id` 和 `created_at` 欄位上建立 [索引](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E4%BD%BF%E7%94%A8%E6%AD%A3%E7%A2%BA%E7%9A%84%E7%B4%A2%E5%BC%95)。

`monthly_spending` 資料表可能包含以下結構：

```
id int NOT NULL AUTO_INCREMENT
month_year date NOT NULL
category varchar(32)
amount decimal NOT NULL
user_id int NOT NULL
PRIMARY KEY(id)
FOREIGN KEY(user_id) REFERENCES users(id)
```

我們可以建立 [索引](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E4%BD%BF%E7%94%A8%E6%AD%A3%E7%A2%BA%E7%9A%84%E7%B4%A2%E5%BC%95) 在 `id` 和 `user_id` 欄位。

#### 類別服務

對於**類別服務**來說，我們可以提供一個「賣家與類別」對應的對應表，當中列出熱門的賣家。如果我們預估有 50000 個賣家，並且每個項目會使用少於 255 bytes，那整個對應表也僅僅需要 12 MB 的記憶體空間。

**向你的面試者詢問他預期你的程式碼要寫到什麼程度**.

```
class DefaultCategories(Enum):

    HOUSING = 0
    FOOD = 1
    GAS = 2
    SHOPPING = 3
    ...

seller_category_map = {}
seller_category_map['Exxon'] = DefaultCategories.GAS
seller_category_map['Target'] = DefaultCategories.SHOPPING
...
```

對於那些最初沒有被挑中的賣家來說，我們可以藉由手動的方式讓使用者覆蓋掉我們的設定。使用 heap 可以在 O(1) 的時間內挑出前幾名的賣家。

```
class Categorizer(object):

    def __init__(self, seller_category_map, self.seller_category_crowd_overrides_map):
        self.seller_category_map = seller_category_map
        self.seller_category_crowd_overrides_map = \
            seller_category_crowd_overrides_map

    def categorize(self, transaction):
        if transaction.seller in self.seller_category_map:
            return self.seller_category_map[transaction.seller]
        elif transaction.seller in self.seller_category_crowd_overrides_map:
            self.seller_category_map[transaction.seller] = \
                self.seller_category_crowd_overrides_map[transaction.seller].peek_min()
            return self.seller_category_map[transaction.seller]
        return None
```

交易的實作部分：

```
class Transaction(object):

    def __init__(self, created_at, seller, amount):
        self.timestamp = timestamp
        self.seller = seller
        self.amount = amount
```

### Use case: Service recommends a budget

To start, we could use a generic budget template that allocates category amounts based on income tiers.  Using this approach, we would not have to store the 100 million budget items identified in the constraints, only those that the user overrides.  If a user overrides a budget category, which we could store the override in the `TABLE budget_overrides`.

```
class Budget(object):

    def __init__(self, income):
        self.income = income
        self.categories_to_budget_map = self.create_budget_template()

    def create_budget_template(self):
        return {
            'DefaultCategories.HOUSING': income * .4,
            'DefaultCategories.FOOD': income * .2
            'DefaultCategories.GAS': income * .1,
            'DefaultCategories.SHOPPING': income * .2
            ...
        }

    def override_category_budget(self, category, amount):
        self.categories_to_budget_map[category] = amount
```

For the **Budget Service**, we can potentially run SQL queries on the `transactions` table to generate the `monthly_spending` aggregate table.  The `monthly_spending` table would likely have much fewer rows than the total 5 billion transactions, since users typically have many transactions per month.

As an alternative, we can run **MapReduce** jobs on the raw transaction files to:

* Categorize each transaction
* Generate aggregate monthly spending by category

Running analyses on the transaction files could significantly reduce the load on the database.

We could call the **Budget Service** to re-run the analysis if the user updates a category.

**Clarify with your interviewer how much code you are expected to write**.

Sample log file format, tab delimited:

```
user_id   timestamp   seller  amount
```

**MapReduce** implementation:

```
class SpendingByCategory(MRJob):

    def __init__(self, categorizer):
        self.categorizer = categorizer
        self.current_year_month = calc_current_year_month()
        ...

    def calc_current_year_month(self):
        """Return the current year and month."""
        ...

    def extract_year_month(self, timestamp):
        """Return the year and month portions of the timestamp."""
        ...

    def handle_budget_notifications(self, key, total):
        """Call notification API if nearing or exceeded budget."""
        ...

    def mapper(self, _, line):
        """Parse each log line, extract and transform relevant lines.

        Argument line will be of the form:

        user_id   timestamp   seller  amount

        Using the categorizer to convert seller to category,
        emit key value pairs of the form:

        (user_id, 2016-01, shopping), 25
        (user_id, 2016-01, shopping), 100
        (user_id, 2016-01, gas), 50
        """
        user_id, timestamp, seller, amount = line.split('\t')
        category = self.categorizer.categorize(seller)
        period = self.extract_year_month(timestamp)
        if period == self.current_year_month:
            yield (user_id, period, category), amount

    def reducer(self, key, value):
        """Sum values for each key.

        (user_id, 2016-01, shopping), 125
        (user_id, 2016-01, gas), 50
        """
        total = sum(values)
        yield key, sum(values)
```

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/V5q57vU.png)

**Important: Do not simply jump right into the final design from the initial design!**

State you would 1) **Benchmark/Load Test**, 2) **Profile** for bottlenecks 3) address bottlenecks while evaluating alternatives and trade-offs, and 4) repeat.  See [Design a system that scales to millions of users on AWS](../scaling_aws/README.md) as a sample on how to iteratively scale the initial design.

It's important to discuss what bottlenecks you might encounter with the initial design and how you might address each of them.  For example, what issues are addressed by adding a **Load Balancer** with multiple **Web Servers**?  **CDN**?  **Master-Slave Replicas**?  What are the alternatives and **Trade-Offs** for each?

We'll introduce some components to complete the design and to address scalability issues.  Internal load balancers are not shown to reduce clutter.

*To avoid repeating discussions*, refer to the following [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) for main talking points, tradeoffs, and alternatives:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
* [Horizontal scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Web server (reverse proxy)](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [API server (application layer)](https://github.com/donnemartin/system-design-primer#application-layer)
* [Cache](https://github.com/donnemartin/system-design-primer#cache)
* [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL write master-slave failover](https://github.com/donnemartin/system-design-primer#fail-over)
* [Master-slave replication](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Asynchronism](https://github.com/donnemartin/system-design-primer#asynchronism)
* [Consistency patterns](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [Availability patterns](https://github.com/donnemartin/system-design-primer#availability-patterns)

We'll add an additional use case: **User** accesses summaries and transactions.

User sessions, aggregate stats by category, and recent transactions could be placed in a **Memory Cache** such as Redis or Memcached.

* The **Client** sends a read request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
    * Static content can be served from the **Object Store** such as S3, which is cached on the **CDN**
* The **Read API** server does the following:
    * Checks the **Memory Cache** for the content
        * If the url is in the **Memory Cache**, returns the cached contents
        * Else
            * If the url is in the **SQL Database**, fetches the contents
                * Updates the **Memory Cache** with the contents

Refer to [When to update the cache](https://github.com/donnemartin/system-design-primer#when-to-update-the-cache) for tradeoffs and alternatives.  The approach above describes [cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside).

Instead of keeping the `monthly_spending` aggregate table in the **SQL Database**, we could create a separate **Analytics Database** using a data warehousing solution such as Amazon Redshift or Google BigQuery.

We might only want to store a month of `transactions` data in the database, while storing the rest in a data warehouse or in an **Object Store**.  An **Object Store** such as Amazon S3 can comfortably handle the constraint of 250 GB of new content per month.

To address the 2,000 *average* read requests per second (higher at peak), traffic for popular content should be handled by the **Memory Cache** instead of the database.  The **Memory Cache** is also useful for handling the unevenly distributed traffic and traffic spikes.  The **SQL Read Replicas** should be able to handle the cache misses, as long as the replicas are not bogged down with replicating writes.

200 *average* transaction writes per second (higher at peak) might be tough for a single **SQL Write Master-Slave**.  We might need to employ additional SQL scaling patterns:

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

We should also consider moving some data to a **NoSQL Database**.

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process
