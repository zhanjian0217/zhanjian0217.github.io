---
author: 濟安
pubDatetime: 2023-09-17
title: 安裝 PostgreSQL
postSlug: how-to-install-postgres
featured: false
draft: false
tags:
  - database
description: 如何安裝 Postgresql
---

昨天我們大致介紹了什麼是 SQL & RDBMS，今天就讓我們來安裝好我們的環境，之後才能開始使用 SQL 語法

### 下載以及安裝 PostgreSQL (本文以 macOS 示範)
在 macOS 中主要可以使用兩種方式進行下載


1. 使用 Homebrew 安裝
2. 從 PostgreSQL 官方網站安裝

* *本文中出現的 `$` 代表後面的指令需要在終端機輸入*



## 使用 Homebrew 安裝



1. 先確認是否有安裝 `Homebrew` 

```shell
$ brew --version
```
若是已經安裝好了，正常來說的終端機回應會像是下面這樣

```shell
Homebrew 4.1.6-16-g3c8b494
Homebrew/homebrew-core (git revision 3c144eb3f0c; last commit 2023-08-07)
Homebrew/homebrew-cask (git revision af0d094733; last commit 2023-08-06)
```

若是還沒有安裝好的那就必須先去 [Homebrew 官方網站](https://brew.sh/) 安裝 Homebrew


2. 使用 brew 指令安裝 postgresql 並啟動

```shell
$ brew install postgresql@15 # 本文以 15 版本作為範例

.......
==> Summary
🍺  /usr/local/Cellar/postgresql@15/15.4: 3,698 files, 60.6MB
==> Running `brew cleanup postgresql@15`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).

# 若是成功的話，會看到一個啤酒杯
```

緊接的來啟動 PostgreSQL 服務

```shell
$ brew services start postgresql@15

==> Successfully started `postgresql@15` (label: homebrew.mxcl.postgresql@15)
```

來確認一下是否有啟動 PostgreSQL 服務

```shell
$ brew services list

Name          Status
postgresql@15 started
```

若是不放心的話，我們可以用 lsof 指令再次確認

*`lsof` 的 `-i` 參數可以用來查詢本機所有的網路連線*

```shell
$ lsof -i:5432 # postgresql 預設的 port 為 5432

COMMAND    PID USER      FD  TYPE .....
postgres 26613 yourname  7u  IPv6 .....
postgres 26613 yourname  8u  IPv4 .....

# 只要有看到 postgres 就是成功的
```

```shell
$ psql --version

#=> psql (PostgreSQL) 15.0
```

## 使用 Postgres.app 官方網站安裝



1. 首先先到 [PostgreSQL](https://www.postgresql.org/) 官方網站，並點選左上方的 `download`

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148nRea9rqUgC.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148nRea9rqUgC.png)

2. 選擇你電腦的系統，這邊我們就選擇 macOs

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148suCyLcoRzS.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148suCyLcoRzS.png)

3. 這邊會提供許多方式下載 PostgreSQL 包括我們剛剛提到的 homebrew 下載，這邊我們就使用 [Postgres.app](https://postgresapp.com/) 下載

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148BOBKTJvQpS.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148BOBKTJvQpS.png)

4. 來到 [Postgres.app](https://postgresapp.com/) 我們就可以選擇想要下載的版本

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148Vqxsz0zKhe.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148Vqxsz0zKhe.png)


5. 依照安裝步驟後我們的電腦就會出現這個應用程式

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148yuTnJMKl9N.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148yuTnJMKl9N.png)

![https://ithelp.ithome.com.tw/upload/images/20230917/20152148lfWohx4PJo.png](https://ithelp.ithome.com.tw/upload/images/20230917/20152148lfWohx4PJo.png)

6. 一樣確認 PostgreSQL 服務 是否有啟動，也一樣可以使用 `lsof` 指令確認


## 開始使用 psql 指令

`psql` 是 PostgreSQL 的命令列界面，是用來與 PostgreSQL 進行互動的，進入這個互動介面，我們就可以使用 SQL 來對 PostgreSQL 進行操作

```shell
$ psql postgres # 登入 postgres 資料庫（預設不用密碼）

# 會進入和 postgresql 的互動介面
psql (15.0)
Type "help" for help.

postgres=#
```

### SHOW 指令

- `SHOW` 指令可以查看 PostgreSQL 中現在所連上的 database 的各種設定
- 在 `psql` 互動介面時，指令的最後都要加上分號 `;` 才會執行

1. 查看 server 的版本

```shell
postgres=# SHOW server_version;

server_version
----------------
 15.0
(1 row)
```

2. 查看時區的設定

```shell
postgres=# SHOW timezone;

  TimeZone
-------------
 Asia/Taipei
(1 row)
```

3. 查看編碼型態

```shell
postgres=# SHOW server_encoding;

 server_encoding
-----------------
 UTF8
(1 row)
```

4. 查看所有設定

```shell
postgres=# SHOW ALL;
```



## 結語

今天完成了安裝 PostgreSQL，以及使用了 psql 指令進入 PostgreSQL 互動介面，並使用 `SHOW` 這個 SQL 語法來列出 `postgres` 資料庫(預設)的一些資訊
