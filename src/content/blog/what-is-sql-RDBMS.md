---
author: 濟安
pubDatetime: 2022-12-28
title: 什麼是 SQL、RDBMS
postSlug: what-is-sql_RDBMS
featured: false
draft: false
tags:
  - database
description: New feature in AstroPaper v1.4.0, introducing dynamic OG image generation for blog posts.
---


## 什麼是 SQL (結構化查詢語言)


SQL `(Structured Query Language)`，中文則為 `結構化查詢語言` 是一種用於管理 關聯式資料庫管理系統 (RDBMS) 的程式語言。讓使用者可以執行各種數據操作，包括查詢、更新、插入和刪除數據


### SQL 的基本語法：

1. 查詢： `SELECT * FROM table;`
2. 插入： `INSERT INTO table (column1, column2) VALUES (value1, value2);`
3. 更新： `UPDATE table SET column1 = value1 WHERE condition;`
3. 刪除： `DELETE FROM table WHERE condition;`

* 這些語法在後續的章節都會詳細介紹到

那什麼是 **關聯式資料庫管理系統** (RDBMS) 呢？

## 什麼是 RDBMS (關聯式資料庫管理系統)

RDBMS `(Relational Database Management System)` 其中文為 關聯式資料庫管理系統，是一種用於存儲、檢索、定義和管理關聯式資料庫 `(Relational database)` 的軟體系統

### 主要特色

1. **數據結構**：使用 table (資料表) 來組織儲存的資料。每個 table (資料表) 中有多個 Column (欄位) 和多個 Row (列)

    舉例來說，如果有一個 `會員` 的 table (資料表)，它可能像這樣：

    
    
    |          姓名           | 年齡 |    電話    |                  |
    |:-----------------------:|:----:|:----------:|:----------------:|
    |          張三           |  28  | 0932xxxxxx | <-- 這是一個 Row |
    |          李四           |  27  | 0912xxxxxx |                  |
    | 每個直排都是一個 Column |      |            |                  |
    



2. **數據完整性和一致性**：通過設定約束（例如，主鍵（Primary Key）、外鍵（Foreign Key）、唯一性約束（Unique Constraint）等）來確保數據的完整性和一致性。

3. **數據查詢和操作**：提供一種專用的查詢語言，即SQL（Structured Query Language），用於數據檢索和操作。

4. **ACID 特性**：即為 `Atomicity (原子性)`、`Consistency (一致性)`、`Isolation (隔離性)`、`Durability (持久性)`，用來確保資料的一致性以及可靠性




### ACID 特性
因為 ACID 特性 很重要，所以特別詳細介紹一下

**1. 原子性(Atomicity)**： 原子性確保了一個交易（Transaction）中的所有操作只有全部成功，或是全部失敗這兩種選項。這也就是表示，即使只有一個步驟失敗，整個交易都會被回滾（Rollback）


```js
你可以想像成假設你在線上商店購買了一個產品，
這個過程可能包括扣款和更新庫存兩個操作，
如果任何一個操作失敗（例如，扣款成功但庫存更新失敗），
原子性（Atomicity）會確保整個交易被取消，並將已扣款退回
```

```sql
-- 假設有一個名為 orders 的資料表和一個名為 inventory 的資料表

BEGIN;
    -- 扣款
    UPDATE orders SET amount = amount - p_amount WHERE order_id = 1;

    -- 更新庫存
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 100;

    -- 檢查是否有錯誤，例如庫存不足
    IF FOUND THEN
        COMMIT;
    ELSE
        ROLLBACK;
        RAISE 'Error occurred, transaction rolled back';
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;

```
   

**2. 一致性(Consistency)**：確保交易將資料庫從一個一致的狀態轉變到另一個一致的狀態，不會違反任何完整性約束（Integrity Constraints）

```js
你可以想像一個在銀行轉帳的例子，
假如你從一個帳戶轉出 100 元到另一個帳戶，
那麼這個過程必須保持一致性，確保總金額不變
```

```sql
-- 假設有一個銀行帳戶資料表bank_accounts

-- 轉帳
BEGIN
    -- 扣款
    UPDATE bank_accounts SET balance = balance - 100 WHERE account_id = 99;

    -- 存款
    UPDATE bank_accounts SET balance = balance + 100 WHERE account_id = 100;

    -- 檢查一致性約束，例如不允許餘額為負
    IF EXISTS (SELECT * FROM bank_accounts WHERE balance < 0) THEN
        ROLLBACK;
    ELSE
        COMMIT;
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
```

**3. 隔離性(Isolation)**：是指即使多個交易同時進行，每個交易也應該獨立於其他交易。這意味著一個交易的執行不應該被其他交易干擾

```js
假設兩個用戶同時在線上商店購買最後一件特定產品，
隔離性確保這兩個交易不會互相干擾，
只有一個交易會成功完成，而另一個會因為庫存不足而失敗
```

```sql
-- 在SQL Server中，你可以設定隔離級別

-- 設定隔離級別
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 購買最後一件產品
UPDATE inventory SET stock = stock - 1 WHERE product_id = 101 AND stock > 0;

COMMIT;

```

**4. 持久性(Durability)**：確保一旦交易被確認，其變更將永久地保存在資料庫中，即使在系統故障後也不會丟失


```js
在一個訂票系統中，如果你成功購買了一張飛機票，
除非存放空間的硬體受損
不然的話即使系統後來發生故障，你的訂票資訊也會被永久保存，
不會因為故障而消失
```

```sql
BEGIN;

-- 購買飛機票
INSERT INTO tickets (user_id, flight_id) VALUES (1, 1001);

COMMIT;
```


### 常見的RDBMS軟體

* Oracle Database
* Microsoft SQL Server
* MySQL
* PostgreSQL
* IBM Db2
* SQLite


## 結語


今天先介紹什麼是 SQL 以及 RDBMS
明天我們將開始安裝 PostgreSQL，並開啟探索 SQL 世界的大門

*PostgreSQL是一個開源的RDBMS，它是功能豐富和高度擴展性的*
*此系列文都將以 PostgreSQL 作為範例*