---
author: 濟安
pubDatetime: 2023-09-19
title: 建立資料表 (table) & INSERT INTO 語法
postSlug: create-table-and-insert-into
featured: false
draft: false
tags:
  - database
description: 建立一個資料表 (table)，並且將資料寫入這個資料表 (table) 也就是大家可能有聽過的 `INSERT INTO` SQL 指令
---


昨天已經成功建立一個資料庫 (database) 了，今天開始就會建立一個資料表 (table)，並且將資料寫入這個資料表 (table)
也就是大家可能有聽過的 `INSERT INTO` SQL 指令

但是在這之前，我先再來回顧一下資料表 (table)， 資料庫 (database)，以及 關聯式資料庫管理系統 (RDBMS)

- 以我們使用的 `postgresql` 來說他就是 RDBMS，而每個資料庫就是 `關聯式資料庫  Relational Database`

## 資料表 (table) & 資料庫 (database) & 關聯式資料庫管理系統 (RDBMS)

    大家可以看到下圖是一個簡單的 資料表 (table)
    他是由許多的單元格 (cell) 所組成，而這些單元格則是由列 (Row) 和 欄位 (column) 交集所產生的
    
    但是這些單位都必須依靠 資料表 (table) 所產生的
    
![https://ithelp.ithome.com.tw/upload/images/20230918/20152148xwBpoEfzEn.png](https://ithelp.ithome.com.tw/upload/images/20230918/20152148xwBpoEfzEn.png)

    所以我們可以知道一個資料庫是由很多的資料表產生的

![https://ithelp.ithome.com.tw/upload/images/20230918/20152148sLvGUrG4HT.png](https://ithelp.ithome.com.tw/upload/images/20230918/20152148sLvGUrG4HT.png)

    很多的資料庫則被一個 PostgreSQL 服務所管理，形成一個資料庫叢集

![https://ithelp.ithome.com.tw/upload/images/20230918/20152148MeWvSvJtGs.png](https://ithelp.ithome.com.tw/upload/images/20230918/20152148MeWvSvJtGs.png)

所以我們可以知道雖然單元格 (cell) 是最小的單位，但是在 RDBMS 中資料表(table) 是基本的數據組織和存儲單位，由多個字段或單元格組成。

所以就讓我們來建立一個資料表吧

## CREATE TABLE

1. 首先我們先 CREATE DATABASE，並使用 psql 進入 postgresql 終端互動介面

```shell
$ createdb demo_db

$ psql demo_db
```

2. 我們可以創建一個新的資料表 (table)，為它取一個名字，並且宣告所有的欄位名稱與其資料型別

```sql
demo_db=> CREATE TABLE books (
              author        varchar(80),
              title         varchar(80),
              price         int,
              create_at     date
          );

CREATE TABLE
```

- 你可以把上述內容在 psql 中輸入，包含換行字元不會影響判讀。**psql 是以分號作為指令結束的判定**

- 這邊我們以資料表 `books` 為例，通常會以複數為資料表的名字

- `varchar(80)`  是一個資料型別， n 可以為任意數，它可以儲放任意 80 個字元以內的字串，是以 byte 為單位，繁體中文中的一個字不會為一個字，通常是佔 3 ~ 4 個 byte (1byte=8bits) （這邊以 ruby 的 irb 來驗證）

![https://ithelp.ithome.com.tw/upload/images/20230918/20152148X0KufRvcxs.png](https://ithelp.ithome.com.tw/upload/images/20230918/20152148X0KufRvcxs.png)


- `int` 是一般認知的整數型別
- `date` 就是日期時間型別 格式為 `YYYY-MM-DD`


## INSERT INTO

- INSERT INTO 是 SQL 中用於向表中插入新行的指令。它可以用來添加單行或多行數據，並且提供了多種方式來指定要插入的數據。

- 為了排版方便，所以將 `（你的資料庫名稱）=#` 省略，下面的指令都是在 psql 的終端介面指令執行的喔


1. 一次寫入一筆資料

```sql
-- psql

demo_db=> INSERT INTO books (author, title, price, create_at) 
          values ('我是作者', '好看的書', 1000, '2023-09-19');

INSERT 0 1
```

- 這邊我們有看到 postgresql 回應我們 `INSERT 0 1`，這代表資料有成功寫入資料庫了，那我們要怎麼驗證了，讓我們來使用 `SELECT` 語法吧

```sql
demo_db=> SELECT * FROM books;
```

結果：
```sql
  author  |  title   | price | create_at
----------+----------+-------+------------
  我是作者  | 好看的書  | 1000  | 2023-09-19
(1 row)
```

這樣我們就可以成功的查詢到我們剛剛寫入的資料了
但是這樣好像有點不方便，是不是每次寫入資料都要再執行一次命令，好像有點麻煩，而且還要用 SELECT 去驗證一下，好麻煩，其實我們也是可以一次寫入很多比資料的

2. 一次寫入多筆資料

```sql
demo_db=> INSERT INTO books (author, title, price, create_at)
          VALUES 
          ('喬治·歐威爾', '1984', 9.99, '2021-01-01'),
          ('J.K 羅琳', '哈利波特', 12.99, '2022-02-15'),
          ('J·R·R·托爾金', '魔戒', 15.99, '2022-03-10');
```

3. 使用 `RETURNING` 子句

- 用 RETURNING 字句，可以回傳剛剛執行的結果，也可以避免執行額外的資料庫查詢來收集資料，特別是在難以可靠地識別修改的資料列時尤其有用

- 以我們剛剛也寫入一筆有關書的資料為例，我們只要在後面加上 `RETURNING *;` (返回所有列) 就會返回剛剛寫入的資料

```sql

demo_db=> INSERT INTO books (author, title, price, create_at)
          VALUES ('我是作者', '我的書籍', 1999, '2023-09-19')
          RETURNING *;
```

結果：
```sql
  author  |  title   | price | create_at
----------+----------+-------+------------
 我是作者  | 我的書籍   | 1999  | 2023-09-19
(1 row)

INSERT 0 1
``` 

- `RETURNING (特定的欄位 多欄位可以 , 隔開) 返回特定列`
```sql
--sql 

demo_db=> INSERT INTO books (author, title, price, create_at)
          VALUES ('我是作者', '我的書籍', 1999, '2023-09-19')
          RETURNING title, author;

```

結果：
```sql

  title   |  author
----------+----------
 我的書籍   | 我是作者
(1 row)

INSERT 0 1
```


- 返回運算結果，您甚至可以返回某種運算結果。例如，如果您想返回書的價格加稅（假設稅率是 10%），您可以這樣寫：


```sql
demo_db=> INSERT INTO books (author, title, price, create_at)
          VALUES ('作者本人', '好貴的書', 10000, '2021-01-01')
          RETURNING author, title, price * 1.1 AS price_with_tax;
```

結果：
```sql

 author  |  title   | price_with_tax
----------+----------+----------------
 作者本人   | 好貴的書  |     11000.0
(1 row)

INSERT 0 1
```

加上了 `RETURNING`，我們就可以不用在另外使用 `SELECT` 查詢語句了，真是太棒了


## DROP TABLE

`$ DROP TABLE tablename;`

若是你不需要這個資料表了，那麼我們就可以將它刪除了，但是基本上工作的時候不會用到這個指令，所以請小心使用。

那麼就來將剛剛所建立的資料表 `books` 刪除吧

```sql
demo_db=> DROP TABLE books

DROP TABLE
```

## 結語

今天我們一起建立資料表(table)，並且使用 SQL 語法 `INSERT INTO` 將資料寫入資料表中了，也就代表成功的將資料存到資料庫了。
明天我們就要將這些資料讀取出來 ![/images/emoticon/emoticon50.gif](/images/emoticon/emoticon50.gif)



