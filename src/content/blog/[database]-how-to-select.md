---
author: 濟安
pubDatetime: 2023-09-20
title: 資料表 (table) 的查詢
postSlug: how-to-use-select-command
featured: false
draft: false
tags:
  - database
description: 如何使用 Postgresql 指令來查詢資料庫
---

昨天我們成功的使用 `CREATE TABLE` 成功的建立了一個資料表 (table)，並且使用 `INSERT INTO` 這個 SQL 語法成功的將資料寫入我們的資料庫中，雖然 `儲存資料` 就是我們使用資料庫的目的之一，但是別忘了，`讀取資料` 也是我們使用資料庫的目的，所以今天就來介紹如何對資料庫進行查詢


## SELECT 指令

SQL 中的 `SELECT` 指令，基本上是由幾個部分所組成的，回傳列表（select list，想要回傳的欄位）、資料表列表（資料來源的資料表）、選擇性的條件定義（指定一些限制條件）


- 假設我們有個 books 資料表 (table)

| author | title | price | create_at |
| -------- | -------- | --- | -------- |
| J·R·R·托爾金 |    魔戒    |  2000 | 1954-07-29 |
| J.K 羅琳     |  哈利波特  |  1500  | 1998-09-01 |
| 喬治·歐威爾   |    1984  |   999  | 1949-06-08 |


### 1. 查詢所有資料

假設我們想要查詢 books 這個 table 中所有的資料

```sql!
demo_db=> SELECT * FROM books;
```

結果：
```sql
    author    |  title   | price | create_at
--------------+----------+-------+------------
 J·R·R·托爾金 | 魔戒     |  2000 | 1954-07-29
 J.K 羅琳     | 哈利波特 |  1500 | 1998-09-01
 喬治·歐威爾  | 1984     |   999 | 1949-06-08
 
(4 rows)
```

這邊我們所使用的 `*` 代表，所有 `欄位 column` 的意思，所以我們來看看能不能翻譯成中文


```sql!
SELECT * FROM books;

翻譯蒟蒻：選取來自 books 中的所有欄位

-- 備註：沒有附上查詢條件的話就會回傳 table 中的所有 列(row) 
```

這樣看起來感覺挺不賴的


### 2. 指定特定欄位的查詢

選取全部的欄位我們會用 `*` 來表示，那我只想要看到特定的欄位 column 呢?
很簡單只要寫出欄位名稱就好了


- 這邊我只想查詢作者 以及 標題

```sql
demo_db=> SELECT author, title FROM books;
```

結果：
```sql
    author    |  title
--------------+----------
 J·R·R·托爾金 | 魔戒
 J.K 羅琳     | 哈利波特
 喬治·歐威爾   | 1984
(4 rows)
```


### 3. 加入條件的查詢：WHERE 子句

剛剛我們所執行的 `SELECT` 查詢，是針對欄位 column 作條件的控制，差別只在欄位的呈現是不同的，若是 table 中有一百萬筆資料，那就會回傳一百萬筆，但是很多時候，我們不希望取得 table 中的所有資料，這時我們就可以依靠 `WHERE` 子句來幫助我們


- 條件：只想查詢價格超過 1000 的書籍

```sql
demo_db=> SELECT * FROM books WHERE price > 1000;

翻譯蒟蒻： 選取來自 books 中的所有欄位，但是條件是價錢要大於 1000
```

結果：
```sql
    author    |  title   | price | create_at
--------------+----------+-------+------------
 J·R·R·托爾金 | 魔戒     |  2000 | 1954-07-29
 J.K 羅琳     | 哈利波特 |  1500 | 1998-09-01 
(2 rows)
```

    tips: WHERE 的內容是一個布林（truth value）表示式，
    而只有在其運算值為真（true）時，該列才會被回傳。
    一般的布林運算子（AND, OR, NOT）都是被允許出現在表示式中的。


### 4. 複合條件查詢

再讓我們提升一點複雜度

- 條件：查詢書本價錢為 1000 元以上 且 出版日期為 1990 以後

```sql
demo_db=> SELECT * 
          FROM books 
          WHERE price > 1000 
          AND  create_at > '1990-01-01';
```

結果：
```
  author  |  title   | price | create_at
----------+----------+-------+------------
 J.K 羅琳 | 哈利波特 |  1500 | 1998-09-01
(1 row)
```

### 5. 排序查詢結果：ORDER BY 子句

很多時候，我們找到了我們查詢條件的資料，但是我們需要將這些資料依照欄位排序好，這時就可以使用 `ORDER BY` 子句


- 條件： 查詢書本價錢為 1000 元以上 並且 依照價格由高到低排序


```sql
demo_db=> SELECT * 
          FROM books 
          WHERE price > 1000 
          ORDER BY price DESC;
```

結果：

```sql
    author    |  title   | price | create_at
--------------+----------+-------+------------
 J·R·R·托爾金 | 魔戒     |  2000 | 1954-07-29
 J.K 羅琳     | 哈利波特 |  1500 | 1998-09-01
(2 rows)
```

    tips: 按照價格的升序（從高到低）來排序查詢結果。如果你想按照價格的升冪排序，可以使用 ASC：



### 6. 一些常見的條件運算子

#### `IN`：如果列的值存在於一個指定的集合中，則返回 `TRUE`

條件：我的書單有 ('哈利波特', '航海王', '火影忍者')，我希望可以在這個書店中找到

```sql
demo_db=> SELECT *
          FROM books
          WHERE title IN ('哈利波特', '航海王', '火影忍者');
```

結果：
```sql
  author  |  title   | price | create_at
----------+----------+-------+------------
 J.K 羅琳 | 哈利波特 |  1500 | 1998-09-01
(1 row)

看來只能找到一本
```

#### `BETWEEN`：在兩個值之間

條件：找到價格介於 1000 ~ 1999 元的書

```sql
demo_db=> SELECT *
          FROM books
          WHERE price BETWEEN 1000 AND 1999;

or

demo_db=> SELECT *
          FROM books
          WHERE price > 1000 AND price < 1999;
```

結果：
```sql
  author  |  title   | price | create_at
----------+----------+-------+------------
 J.K 羅琳 | 哈利波特 |  1500 | 1998-09-01
(1 row)
```

#### `!=` 或 `<>`：不等於

條件：找到不是 J.K 羅琳 的書

```sql
demo_db=> SELECT *
          FROM books
          WHERE author <> 'J.K 羅琳';
```

結果：
```sql
    author    | title | price | create_at
--------------+-------+-------+------------
 J·R·R·托爾金 | 魔戒  |  2000 | 1954-07-29
 喬治·歐威爾  | 1984  |   999 | 1949-06-08
(2 rows)
```

#### `LIKE`：用於進行模糊匹配

條件：找到作者的姓名是 J 開頭的書

```sql
demo_db=> SELECT *
          FROM books
          WHERE author LIKE 'J%';
```

結果：
```sql
    author    |  title   | price | create_at
--------------+----------+-------+------------
 J·R·R·托爾金 | 魔戒     |  2000 | 1954-07-29
 J.K 羅琳     | 哈利波特 |  1500 | 1998-09-01
(2 rows)
```

結語：

今天使用了 SQL 語法中很重要的 `SELECT` 語法，有很多查詢都是基於這個 `SELECT` 開始的
明天將為大家介紹 `JOINS`
 