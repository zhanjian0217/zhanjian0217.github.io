---
author: 濟安
pubDatetime: 2023-09-18
title: 如何建立資料庫
postSlug: create-db
featured: false
draft: false
tags:
  - database
description: 如何使用 Postgresql 建立資料庫
---


昨天我們安裝好了，`Postgresql` 並且使用了 SHOW 這個 SQL 語法，將 `Postgresql` 的一些基本設定打印出來了，今天開始我們就要開始建立自己的資料庫 (database) 並生成一個資料表 (table)，最後再將它們通通刪掉，就讓我們開始來操作吧



## 使用 `createdb` 命令建立資料庫

- 首先我們可以使用 `postgresql` 提供給我們的指令 `createdb` 建立資料庫
- `createdb` 是 `postgresql` 提供的二進位檔案 (bin)


1. 先查看 createdb 指令是否有存在

```shell
$  which createdb

=> ......../bin/createdb
```

2. 再來就是幫自己的資料庫取個好名字，並輸入下面的指令

- `database_demo`，是我自己取得資料庫名稱，各位可以自行更換喔



```shell
$ createdb database_demo

# 這時會發現終端機，好像沒什麼反應
# 我們可以輸入以下指令來查看是否剛剛的指令有成功執行
# 若是終端機回應 0 代表剛剛的指令有被成功執行嘍
$ echo $?

> 0

# 或是在輸入 creatdb 指令時加上 `-e` 這個參數
$ createdb -e database_demo

> CREATE DATABASE database_demo;
```



3. 再來就是讓我們列出所有的 database 查看

```shell
$ psql -l
```

![https://ithelp.ithome.com.tw/upload/images/20230917/201521487wtZzszwGH.png](https://ithelp.ithome.com.tw/upload/images/20230917/201521487wtZzszwGH.png)


- **Encoding:** 指定我們的資料庫要如何儲存字元（character），通常採用 `UTF8` 的編碼方式

- **Collate:** 決定了資料庫如何進行字串的比較。這會影響到 SQL 查詢中的 ORDER BY 子句，通常會採用 `en_US.UTF-8` 或是 `C`

這邊舉個例子來說明，有個資料表是記錄水果的名稱 

|  name   |
|  ----   |
|  apple  |
|  Banana |
|  cherry |

```sql
CREATE TABLE fruits (name TEXT);
INSERT INTO fruits VALUES ('apple'), ('Banana'), ('cherry');
```


![https://ithelp.ithome.com.tw/upload/images/20230917/20152148KUhIHG8ura.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148KUhIHG8ura.png)


    可以看到我們的第二行指令後面有加上 `COLLATE "C"`

    則排序會根據字節值。在 ASCII 編碼中，大寫字母的字節值比小寫字母小。

    這是因為在 ASCII 編碼中，B（ASCII 值為 66）比 a（ASCII 值為 97）和 c（ASCII 值為 99）的字節值小。


- **Ctype:** 指的是字元分類，會按照最基本的字符分類規則處理字符串，這邊我們只要暫時知道使用 正則表達式 的規則會非常基本就好


## 使用 `CREATE DATABASE`

1. 使用 `postgresql` 預設的使用者進入 psql 終端交互介面

```shell
$ psql -U postgres

或是

$ psql
```

2. 使用 `CREATE DATABASE (你的資料庫名稱);`

```sql
# 要記得在指令的結尾加上分號
postgres=> CREATE DATABASE database_deme;

CREATE DATABASE
```


3. 試著將剛剛建立的 database 列出來

```sql
postgres=> \l 
# 列出全部的 database

postgres=> \l database_deme;
# 或是只列出剛剛建立的 database

postgres=> \l+ database_deme;
# 加上 `+` 可以看到更多資訊
```



## 刪除剛剛我們所建立的 database

1. dropdb 指令

```shell
$ dropdb -e database_demo
# 加上 -e 參數讓我們知道 sql server 有回應我們

DROP DATABASE database_demo;
```

2. 使用 DROP DATABASE

```sql
postgres=> DROP DATABASE database_deme;

DROP DATABASE
```


## 結語

今天分別介紹了在 `postgresql` 中兩種方式去建立資料庫，以及刪除資料庫，也有提到了一些 SQL 語法，今天都把資料庫都建立好了，那明天我們就來做個資料表 (table) 吧!