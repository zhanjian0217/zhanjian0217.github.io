---
pubDatetime: 2026-03-13
title: 讀《The Harness Problem》後，我重新思考 Vibe coding 真正的瓶頸
postSlug: rethinking-vibe-coding-harness-problem
featured: false
draft: false
tags:
  - ai
  - tooling
  - vibe-coding
description: 讀完《The Harness Problem》後，重新思考 AI coding 的瓶頸，可能不只在模型能力，也在 harness 與 edit tool 的設計。
---

原文：<https://blog.can.ac/2026/02/12/the-harness-problem>

## 前言：我原本也以為問題主要在模型

現在談 Vibe coding，大家最常問的幾乎都是「哪個模型比較強」
但這篇文章提醒我，也許很多失敗不只是模型能力問題，而是模型與工作環境之間那層工具設計出了問題
所以這篇文章真正要挑戰的，不是模型重不重要，而是：**我們是不是一直把焦點放錯地方了？**

---

## 原文在談什麼

作者認為，大家太習慣把 Vibe coding 的成敗歸因於 model，本質上像是在比模型排行榜
但在實務上，真正大量影響結果的，常常是 **harness**：也就是模型如何讀檔、改檔、接收錯誤訊息、管理狀態，以及最後怎麼把輸出落實成實際修改

換句話說，問題不一定是模型不懂，而可能是模型根本沒被提供一個足夠穩定的工作方式
作者甚至直接說，很多失敗其實就發生在「模型知道要改什麼」到「問題真的被解掉」之間

## 文章最關鍵的地方：edit tool

作者把問題縮到一個非常具體的工程細節：**edit tool 怎麼設計**

文中對比了幾種常見方式：

- `apply_patch`：要求模型輸出某種 patch 格式，這是 Codex 採用的方式之一
- `str_replace`：要求模型精準找到舊文字再替換，Claude Code 與多數其他工具大致屬於這一類
- `hashline`：替每一行加上 hash tag，讓模型用 tag 來指定修改位置

作者的重點不是單純說某個格式比較厲害，而是指出：
很多 edit tool 都在要求模型「重現舊內容」，這本身就很脆弱
而 hashline 的方向，是把修改從「重背內容」變成「穩定定位」

## benchmark 與結果

這篇文章不是停在概念，而是有做 benchmark。
作者從 React codebase 抽檔案，加入一些可逆、局部的 bug mutation，再讓不同模型搭配不同 edit tool 去修復，最後比對是否成功回到原始正確狀態
像是 operator swap、boolean flip、off-by-one、optional chain removal、identifier rename，都是文中提到的 mutation 類型

結果顯示，**同一個模型，只換 edit format，表現就可能差非常多**
作者想說明的是：有些模型看起來不會寫 code，可能不是因為它真的不會，而是因為它不擅長你要求它使用的那種改檔語言

這也代表我們平常以為自己在比較模型能力，實際上很可能也把工具設計的差異一起算進去了

---

## 什麼是 harness

我自己的理解是，harness 就像是 **模型與真實工作環境之間的協議層**
像 `Claude Code`、`Codex`、`Gemini CLI` 這類 coding agent / CLI 工具，實際上都各自帶有自己的 harness 設計

模型不是直接操作檔案，而是透過某種工具介面在工作
所以使用者最後看到的，不是模型裸露的能力，而是「模型能力經過 harness 之後的結果」

這也代表一件事：如果 harness 很脆弱，那模型再強，實際表現也可能被拖垮

## 真正重點是表達失敗

我覺得這篇文章最值得帶走的，不是 hashline 本身，而是它重新切開了「失敗」這件事

很多 Vibe coding 的失敗，不一定是 **理解失敗**，而可能是 **表達失敗**
模型可能知道哪裡有 bug，也知道應該怎麼修，但它沒辦法用工具要求的格式，穩定地把修改表達出來
原文甚至直接說：很多時候，模型不是在理解任務時出問題，而是在表達修改時出問題

所以有時候我們以為自己在評估模型能力，實際上評估到的，是模型加上工具設計的綜合結果

## 為什麼這對工程師重要

這篇文章對工程師特別有啟發，因為 harness 是可以被工程化優化的
大多數工程師沒辦法參與模型訓練，但卻可以參與 tool schema、workflow、error handling、state management 這些設計

如果 harness 的改善真的能帶來接近 model upgrade 的收益，那就代表 Vibe coding 的進步，不一定只來自更大的模型，也來自更紮實的工具工程
作者甚至明講，Gemini 成功率提升 8%，這樣的幅度比很多 model upgrade 還大，而且不需要任何 training compute

## 我自己的保留

我很認同這篇文章提出的問題意識，但我不會把它讀成「模型不重要」
比較準確的說法應該是：**模型很重要，但不是唯一變數**

另外，原文 benchmark 偏向局部、可逆、機械式的 bug 修復
這類任務很適合測 edit format，但如果今天是跨檔案 refactor、架構調整，hashline 是否還有一樣的優勢，我認為還需要更多驗證
文中的 benchmark 本身也主要是以 React codebase 的單檔 mutation 為主，而不是更高層次的架構性修改

## 結語中的判斷

讀完這篇文章後，我最大的改變，不是相信某一種 edit tool 才是正解

而是重新意識到：**AI coding 的競爭，不只是比誰的模型更聰明，也是比誰把模型接到真實工作環境的方式設計得更好**

從 demo 到真正可靠的工具，中間差的可能不是神奇的模型飛躍，而是很多看起來不耀眼、但非常關鍵的工程細節
這也讓我開始覺得，也許接下來值得被認真討論的，不只是 `Prompt engineering`，還有 `Harness engineering`

如果過去大家一直在談 `Prompt engineering`，那接下來 Vibe coding 更值得投入的方向，可能會是 `Harness engineering`
