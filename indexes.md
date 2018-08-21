# Indexes

indexes 是用來增加mongoDB的執行query效能。如果沒有index mongoDB 會掃資料庫的每一個檔案來相配query的需求。如果index有存在，MongoDB就會限制document的數量
去掃描。

### 問題：
假如要在DB裡面查詢first_name為john的資料：
```
db.customers.find({ first_name:"john"}).explain("executionStats")
```
看一下query的細節：
```
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "mycustomers.customers",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"first_name" : {
				"$eq" : "john"
			}
		},
		"winningPlan" : {
			"stage" : "COLLSCAN",
			"filter" : {
				"first_name" : {
					"$eq" : "john"
				}
			},
			"direction" : "forward"
		},
		"rejectedPlans" : [ ]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 0,
		"totalKeysExamined" : 0,
		"totalDocsExamined" : 42,
		"executionStages" : {
			"stage" : "COLLSCAN",
			"filter" : {
				"first_name" : {
					"$eq" : "john"
				}
			},
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 0,
			"works" : 44,
			"advanced" : 1,
			"needTime" : 42,
			"needYield" : 0,
			"saveState" : 0,
			"restoreState" : 0,
			"isEOF" : 1,
			"invalidates" : 0,
			"direction" : "forward",
			"docsExamined" : 42
		}
	},
	"serverInfo" : {
		"host" : "localhost",
		"port" : 27017,
		"version" : "4.0.0",
		"gitVersion" : "3b07af3d4f471ae89e8186d33bbb1d5259597d51"
	},
	"ok" : 1
}

```
可以看到**totalDocsExamined=42** , 代表的是MONGODB掃描了42筆資料（全部）來做這個query。就是說如果裡面有50000筆資料，他會掃50000.

##加index的效果

把first_name加入index：

```
db.customers.createIndex({first_name: 1})
```
接下來我們可以在看一下query的細節：
```
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 0,
		"totalKeysExamined" : 1,
		"totalDocsExamined" : 1,
		"executionStages" : {
			"stage" : "FETCH",
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 0,
			"works" : 2,
			"advanced" : 1,
			"needTime" : 0,
			"needYield" : 0,
			"saveState" : 0,
			"restoreState" : 0,
			"isEOF" : 1,
			"invalidates" : 0,
			"docsExamined" : 1,
			"alreadyHasObj" : 0,
			"inputStage" : {
				"stage" : "IXSCAN",
				"nReturned" : 1,
				"executionTimeMillisEstimate" : 0,
				"works" : 2,
				"advanced" : 1,
				"needTime" : 0,
				"needYield" : 0,
				"saveState" : 0,
				"restoreState" : 0,
				"isEOF" : 1,
				"invalidates" : 0,
				"keyPattern" : {
					"first_name" : 1
				},
				"indexName" : "first_name_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"first_name" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"first_name" : [
						"[\"john\", \"john\"]"
					]
				},
				"keysExamined" : 1,
				"seeks" : 1,
				"dupsTested" : 0,
				"dupsDropped" : 0,
				"seenInvalidated" : 0
			}
		}
	},
  
  ```
  
  現在可以看到**totalDocsExamined=1**，表示MongoDB直接掃描那一筆有**first_name=john**的資料。
  
 ### default _id index
  
創建collection的時候mongodb會幫你建一個唯一的index在id field。基本上 _id index 會預防 clients 輸入兩個有一樣的_id值的文件。 _id index不能被刪除。
  
  ## 創建index
  
  要創建index的指令時：
  ```
  db.collection.createIndex( <key and index type specification>, <options> )
  
  ```
  範例:建個單一降序key的index在name領域
  
  ```
  db.collection.createIndex( { name: -1 } )
  ```
  
  ## index 種類
  
  mongoDB 提供各種不同的index種類來處理不同的query跟data。以下single field, compound index, 會描述的比較細，其他的會做一個比較膚淺的描述。
 
  ### Single Field
  
  這種索引是使用者可以建的，一次可以索引一個領域。
  - 假如有個collection 命名為 records 抱有以下的資料：
 ```
 {
  "_id": ObjectId("570c04a4ad233577f97dc459"),
  "score": 1034,
  "location": { state: "NY", city: "New York" }
}
 ``` 
 以下的指令建個升序的index：
 ```
 db.records.createIndex( { score: 1 } )
 ```
 
 數字是代表那是給那個field的哪種index。例如，1 是代表升序排序回傳的資料，-1是代表降序。
 被創建的index會提供在score領域作select的動作的query，像是：

```
db.records.find( { score: 2 } )
db.records.find( { score: { $gt: 10 } } )
```
#### 在嵌入式field建index

你可以在嵌入式的資料領域建index，也可以指頂級的領域。
- 假如有個collection 命名為 records 抱有以下的資料：
```
{
  "_id": ObjectId("570c04a4ad233577f97dc459"),
  "score": 1034,
  "location": { state: "NY", city: "New York" }
}
```

以下的指令建個index在location.state領域。
```
db.records.createIndex( { "location.state": 1 } )
```
被創建的index會提供在location.state領域作select的query，像是：

```
db.records.find( { "location.state": "CA" } )
db.records.find( { "location.city": "Albany", "location.state": "NY" } )
```
#### 在嵌入式資料建index

你也可以在整個嵌入式的資料建index。
- 假如有個collection 命名為 records 抱有以下的資料：
```
{
  "_id": ObjectId("570c04a4ad233577f97dc459"),
  "score": 1034,
  "location": { state: "NY", city: "New York" }
}
```

location是個嵌入式的資料，裡面有綁定city跟state。以下的指令建個index在location的整個field。

```
db.records.createIndex( { location: 1 } )
```
被創建的index會提供在location領域作select的動作的query，像是：
```
db.records.find( { location: { city: "New York", state: "NY" } } )
```
  
  ### Compound field
  
  如果single field 不達到使用者的需求，compound index 可以指多數的領域。例如，如果有個compound index是 { userid: 1, score: -1 }，指標就會重userid排序然後在每個userid的值內，排序score。
  
  #### 建compound index
 
 以下是compound index的指令原型：
 ```
 db.collection.createIndex( { <field1>: <type>, <field2>: <type2>, ... } )
 ```
 以下有個collection命名為products包以下的資料：
 
 ```
 {
 "_id": ObjectId(...),
 "item": "Banana",
 "category": ["food", "produce", "grocery"],
 "location": "4th Street Store",
 "stock": 4,
 "type": "cases"
}
```
以下的指令會建個升序的index在item跟stock領域：
```
db.products.createIndex( { "item": 1, "stock": 1 } )
```

領域list的順序很重要。index會先排序item領域的value，然後在item field的每個直，排序stock field的直。被創建的index會提供在item領域或item跟stock一起領域的query，像是：
```
db.products.find( { item: "Banana" } )
db.products.find( { item: "Banana", stock: { $gt: 5 } } )
```
#### 排序順序

indexes以升序（1）或降序（-1）排序。對single-field index， key的排序的順序不會影響因為MONGODB可以橫過index的每一邊。不過在compound index排序順序會影響到他是否可以提供sort的動作。
以下有個collection 命名為 events， 有個document包username 跟 date 的領域。應用程式可以執行回傳username得值先升序排序然後在date的值降序的query，像：
```
db.events.find().sort( { username: 1, date: -1 } )
```

或是username 先降序排序在升序date。

```
db.events.find().sort( { username: -1, date: 1 } )
```

以下的index都可以提供這兩種排序的指令：

```
db.events.createIndex( { "username" : 1, "date" : -1 } )
```

但絕對不能提供這種username跟date都升序的：

```
db.events.find().sort( { username: 1, date: 1 } )
```

#### Prefixes

index 的prefixes 是被指標的field subset的「開頭」。例如以下的compound index：
```
{ "item": 1, "location": 1, "stock": 1 }
```
這個index的prefixes就是：

- { item: 1 }
- { item: 1, location: 1 }

MONGODB 可利用index在index prefixes作query，以下的field可以利用index做query的動作。

- item field,
- item field 跟 location field,
- item field 跟 location field 跟 the stock field.

MONGODB 也可以利用index在 item 跟 stock 來作query，因為item定義成個prefix。

但mongoDB不能提供query在以下的field，因為沒有item 的field，其他的都沒被定義成prefix：

- location field,
- stock field, 或
- location and stock fields.
 
### Multikey Index
  
  是用來指標有array的field。如果你索引一個array 的 field，mongoDB會建分開的index給每個array的元素。
  
### Geospatial Index

為了達成有效率的地理空間作標數據的query，MONGODB提供兩種特別的index： 2d indexes會利用平面幾何來呈現結果， 2dsphere indexes 利用球面幾何來呈現結果。

### Text Indexes

mongoDB提供文件索引來尋找collection裡面的參數。


### Hashed Index 
- 雜湊索引會把輸入被索引的field 作雜湊的動作。
- 雜湊索引提供sharding的方法（跨多台機器分發數據的方法）hashed shard 的 key

#### Hashing 的 Function
雜湊索引利用hashing 的 function 來計算雜湊的index field 的直 。 Hashing 沒提供 multikey的index（字串）。
#### 建hashed key
以下的範例：
```
db.collection.createIndex( { _id: "hashed" } )
```
## index 的屬性

### TTL　INDEX 
TTL 索引會照著使用者設定的時間之後自動刪除文件。數據期限對某些資訊種類是很有用的，例如機器產生的event data，logs 還有資訊的session，他們存在database都有限制時間。


要建個TTL 索引可下db.collection.createIndex()方法在加上expireAfterSeconds的選定在領域，裡面的直必須是個DATE 或者是date value 的陣列。

例如：以下會在eventLog的collection建個TTL 索引在lastModifiedDate的field的指令：

```
db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 3600 } )
```

#### 限制
- TTL 索引是single field的索引，compound 索引不提供TTL的功能，他會忽視expireAfterSeconds的選定。
- _id field不提供TTL索引
- 你不能在個 [capped collection](https://docs.mongodb.com/manual/core/capped-collections/) 建TTL索引因為mongoDB不能在 capped collection 刪除文件。
- 你不能用createIndex()來改變已經存在的索引的expireAfterSeconds。請利用 collMod database的指令，或者是刪除目前的索引然後重新建。
- 如果一個不是TTL的索引已存在一個field，你不能在這個field建TTL。要改變這個部分請先刪除目前的index然後在重建個索引加上expireAfterSeconds的選定。

### Unique indexes
Unique index會確保被索引的field不會有一樣的直。

#### 建unique index
 下db.collection.createIndex()在加上unique的選定：
```
db.collection.createIndex( <key and index type specification>, { unique: true } )

```

##### single field 的unique index

以下我們在members的collection裡建個unique索引在user_id：
```
db.members.createIndex( { "user_id": 1 }, { unique: true } )
```

##### compound field 的 unique index

以下我們在groupNumber, lastname, firstname 建個unique索引：

db.members.createIndex( { groupNumber: 1, lastname: 1, firstname: 1 }, { unique: true } )

再舉一個例子：
```
{ _id: 1, a: [ { loc: "A", qty: 5 }, { qty: 10 } ] }
```
建個unique compound的multikey索引在a.loc 跟 a.qty:

```
db.collection.createIndex( { "a.loc": 1, "a.qty": 1 }, { unique: true } )
```

創建的unique索引能允許以下資料的插入到collection因為這個索引強制 a.loc 跟 a.qty 的組合獨特性。
```
db.collection.insert( { _id: 2, a: [ { loc: "A" }, { qty: 5 } ] } )
db.collection.insert( { _id: 3, a: [ { loc: "A", qty: 10 } ] } )
```

### Partial Indexes
它只會索引符合過濾expression的文件。partial 索引會有比較低的存儲要求跟少的績效成本來作索引創建跟維護的動作因為它會索引文件的子集。

#### 建partial index
下db.collection.createIndex()的方法加上partialFilterExpression的選定。partialFilterExpression 可接受以下的條件：

- equality 的expressions(例如：field： value 或是用 $eq 運算符）
- $exists: true 的expression,
- $gt, $gte, $lt, $lte 的 expressions
- $type expressions,
- $and 運算符用在頂層

例如，以下的的指令是來索引文件裡面的大於5的rating:
```
db.restaurants.createIndex(
   { cuisine: 1, name: 1 },
   { partialFilterExpression: { rating: { $gt: 5 } } }
)
```

### Sparse index
跟Partial index很像但sparse索引只會在有FIELD的document建索引key。


## 在大型collection操作index

預設中，如果在個很多資料的collection建索引其他的預算就會被鎖住。在索引沒被建完之前資料庫作讀跟寫的動作。

#### 後台建設
如果有可能長時間運行的索引創建操作，請考慮後台創建的選定，這樣在index創建的進行中mongoDB才仍然可用。

以下的範例在zipcode的後台建個索引。

```
db.people.createIndex( { zipcode: 1 }, { background: true } )
```
你也可以和其他的參數結合：

```
db.people.createIndex( { zipcode: 1 }, { background: true, sparse: true } )
```

## 控管index

### 查詢存在的index

要顯示collection裡所有的index，請下db.collection.getIndexes() ，例如以下的指令會回傳people collection 所有的索引：
```
db.people.getIndexes()
```

顯示資料庫所有的index的指令：

```
db.getCollectionNames().forEach(function(collection) {
   indexes = db[collection].getIndexes();
   print("Indexes for " + collection + ":");
   printjson(indexes);
});
```
### 刪除index

要刪除索引請下db.collection.dropIndex()，如：

```
db.accounts.dropIndex( { "tax-id": 1 } )

```

以下會回傳這個操作的狀態：

```
{ "nIndexesWas" : 3, "ok" : 1 }
```

### 刪除所有的index

以下的指令除了_id以外會刪除collection所有的index：
```
db.accounts.dropIndexes()
```

## 索引參考

#### 索引的方法

|名稱|描述 |
|---|---|
|  db.collection.createIndex() |在collection建個index|
|  db.collection.dropIndex()|在collection刪除某個索引|
|  db.collection.dropIndexes()|在collection刪除所有的索引|
|  db.collection.getIndexes()|回傳個文件的陣列描述collection裡有存在索引|
|  db.collection.reIndex()| 重新建collection裡所有的索引 |
|  db.collection.totalIndexSize()| 回報在collection 裡所有index的總大小 | 
|  cursor.explain()| 回報游標的query執行計劃| 
|  cursor.hint()| 強迫MongoDB在query用特定的索引| 
|  cursor.max()| 指定游標的獨占上限索引，跟cursor.max一起用 | 
|  cursor.min()| 指定游標的包含性較低索引範圍，跟cursor.max一起用 | 

#### DATABASE的索引指令
|名稱|描述 |
|---|---|
|  createIndexes| 創建單一或多數的索引在collection | 
|  dropIndexes| 在collection刪除索引 | 
|  compact| 重新整理collection跟從新創建索引 | 
|  reIndex| 在collection從新創建索引 | 
|  validate| 掃描collection裡的data跟索引尋找正確性的內建指令 | 
|  geoSearch| 做個地理空間的query | 
|  checkShardingIndex| 驗證shard key的index | 

#### 索引query的修飾符

|名稱|描述 |
|---|---|
|  $explain| 強制MongoDB報告查詢執行計劃 | 
|  $hint| 強制MongoDB使用特定索引 | 
|  $max| 指定要在查詢中使用的索引的獨占上限 | 
|  $min| 指定要在查詢中使用的索引的包含性下限| 
|  $returnKey| 強制游標回傳索引裡的field。 | 

