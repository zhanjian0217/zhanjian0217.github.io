---
pubDatetime: 2026-03-26
title: 我用 mitmproxy 看了四個 Code Agent Harness 的底層流量：一些比想像中合理，另一些則比想像中更微妙
postSlug: mitmproxy-code-agent-harness-traffic-analysis
featured: false
draft: true
tags:
  - ai
  - claude-code
  - mitmproxy
description: 用 mitmproxy 攔截 Claude Code、Codex、OpenCode、Pi 的 HTTP/WSS 流量，直接觀察它們怎麼傳資料、維持上下文、掛載工具呼叫
---

# 我用 `mitmproxy` 看了四個 Code Agent Harness 的底層流量：一些比想像中合理，另一些則比想像中更微妙

這篇算是我的調查筆記

最近我花了一點時間，用 `mitmproxy` 去攔截幾個 code agent harness 的 `HTTP` / `WSS` 流量，想直接看它們在傳什麼、怎麼維持上下文、工具呼叫怎麼掛進去
觀察對象是四個：`Claude Code`、`Codex`、`OpenCode`、`Pi`

我做這件事的動機很單純：我想知道，為什麼有些 harness 用起來反應比較快，有些比較慢；為什麼有些明明功能很多，但 token 也明顯燒得比較兇
至於 `Pi`，則是我特別拉進來驗證一個常見說法：它到底是不是真的比較輕量、比較省 token

先講結論：有些現象確實跟我原本猜的一樣，但也有幾個地方比我想得更有意思。尤其當我把 transport、prompt 大小、工具數量一起放進來看之後，會發現「快」跟「省」其實都不是單一因素決定的

## 我怎麼觀察：用 `mitmproxy` 把 `HTTP` / `WSS` 流量攔下來

方法上沒有太神祕，就是把本機代理指到 `mitmproxy`，讓這幾個 harness 經過代理送出請求，然後觀察：

- 它們是走 `HTTPS streaming` 還是 `WebSocket`
- 初始化請求裡帶了多少 system / developer 級別指令
- 每輪互動是整包重送，還是偏增量延續
- 工具宣告有多少個
- 額外的小任務，例如標題生成，是不是另開模型處理

我主要看的是協定型態、payload 結構、prompt 大小量級、工具暴露面，沒有去做嚴格 benchmark，也不是在測端到端的延遲毫秒數。所以這篇比較接近架構觀察，而不是性能評測報告

這點要先說清楚：流量能幫我理解設計取捨，但不能單靠它就斷言「A 一定比 B 快」。因為真正的延遲，還會被模型推理時間、工具執行時間、網路品質、快取命中與否一起影響

## 先放結論表

| Harness       | 傳輸方式                         |                                         系統/指令大小 | 工具數量 | 重點                                        |
| ------------- | -------------------------------- | ----------------------------------------------------: | -------: | ------------------------------------------- |
| `Pi`          | `HTTPS streaming`                |                                         約 `3K chars` |      `4` | 最輕量，省 token 符合預期                   |
| `OpenCode`    | `HTTPS streaming`                |                                         約 `8K chars` |   約 `9` | 比較克制                                    |
| `Claude Code` | `HTTPS streaming`                |      約 `12K chars`（系統層）+ user message reminders |     `28` | 功能面最完整之一，prompt 與工具面也最大之一 |
| `Codex`       | `WebSocket`（偏 `Realtime API`） | 約 `14.6K chars`（指令層）+ 約 `5K` developer message |     `12` | 走長連線與增量延續路線，設計最不一樣        |

## 發現一：只有 `Codex` 走 `WebSocket`，其他三個都還是 `HTTPS streaming`

四個裡面，最醒目的差異是 `Codex`

`Claude Code`、`OpenCode`、`Pi` 都是比較常見的 `HTTPS streaming` 模式；只有 `Codex` 走 `WebSocket`，而且整體感覺很像是以 `Realtime API` 那種長連線思路在設計。這件事代表什麼？代表它比較像是在為長時間、多輪、工具呼叫頻繁的 agent workflow 做最佳化

`WebSocket` 在這種情境的好處很明顯：

- 連線持久，不用每輪都重新建立
- 每輪只送增量輸入，而不是整包上下文重送
- 可以透過像 `previous_response_id` 這種機制延續前一輪狀態
- 如果 active connection 上有 connection-local 的 in-memory cache，可以減少一些重複負擔

如果只看架構，我會說這比傳統 request/response 更像「真正的 agent 執行通道」，而不是把 agent 假裝成聊天室

但這裡有個很容易誤判的地方：`WebSocket` 不等於體感一定快

我自己原本也直覺以為，既然用的是 `WebSocket` 而不是傳統 request/response，反應應該會更俐落。可實際上，體感快慢未必跟這個完全正相關。原因是：

- 單一 socket 本身不能神奇解決所有排程問題
- 真正耗時的常常不是 transport，而是模型推理
- tool call-heavy 的流程，慢點很多時候在工具執行，不在傳輸
- 如果沒有命中 active-connection cache，`WebSocket` 的一部分優勢就不明顯了

所以我的理解是：`Codex` 的 `WebSocket` 設計，不是為了每一輪都「看起來秒回」，而是為了讓長時間、多工具、多輪延續的 workflow 更自然。這是架構上的取捨，不是簡單的速度保證

## 發現二：`Claude Code` 會用小模型處理標題生成

這是一個我很喜歡的小細節

我在流量裡看到，`Claude Code` 在生成對話標題時，用的不是主模型，而是 `claude-haiku-4-5-20251001`。也就是說，標題不是交給主 agent 順手產生，而是切一個更便宜的小模型專門處理

這樣做有幾個好處：

- 標題任務本來就低風險，沒必要用大模型
- 可以省 token
- 可以省錢
- 也能避免主模型把注意力浪費在不重要的 UI 附屬任務上

這種設計看起來不起眼，但我覺得很能反映產品成熟度。因為真正做產品，不是把所有事都丟給最強模型，而是知道哪些地方值得花錢，哪些地方應該精算。標題生成就是很典型的例子：需要自然語言能力，但不需要太強的推理深度，用 `claude-haiku-4-5-20251001` 這種等級處理，其實就很合理

## 發現三：`Pi` 的輕量化，基本上是可觀測、可解釋的

這次我特別把 `Pi` 放進來，某種程度上就是想驗證它「輕量、省 token」到底是真的架構結果，還是只是社群印象

結論很直接：至少從我看到的 prompt 與工具暴露面來說，是真的

`Pi` 的系統提示詞大約只有 `3K chars`，工具只有 `4` 個。這個數字拿來跟其他三個比，非常明顯：

- `Pi`：約 `3K chars`，`4` 個工具
- `OpenCode`：約 `8K chars`，約 `9` 個工具
- `Claude Code`：約 `12K chars`，再加上 user message 裡的 reminders，`28` 個工具
- `Codex`：約 `14.6K chars` 指令層，再加上約 `5K` developer message，`12` 個工具

如果只從 token 經濟學來看，`Pi` 的優勢其實不神祕。它就是：

- 系統提示詞比較短
- 暴露工具比較少
- 功能邊界比較窄

這三件事合起來，本來就會讓它每輪互動更輕。你可以把它理解成：它不是在同樣功能密度下神奇省 token，而是在比較克制的能力配置下，自然達成較低成本

這也代表另一件事：輕量不是免費午餐。當一個 harness 功能比較少、工具比較少時，它當然比較容易省；但相對地，它能處理的複雜 workflow 也可能比較有限。這不是缺點，只是取捨

## 我最後的理解：快慢、省不省，不是單一答案

看完這輪流量之後，我自己對這些 harness 的理解變得比較不浪漫，也比較務實

如果只問「哪個比較快」，答案其實不會只寫在協定名稱上。`WebSocket` 很重要，但它不是萬靈丹；`HTTPS streaming` 比較傳統，也不代表一定慢。真正決定體感的，是：

- transport 設計
- prompt 大小
- 工具數量
- 是否能做增量延續
- cache 有沒有命中
- 模型推理速度
- 工具執行時間

如果只問「哪個比較省 token」，答案也不神祕。很多時候就是 prompt 比較短、工具比較少、任務切分比較精打細算，例如把標題生成交給 `claude-haiku-4-5-20251001` 這種小模型

所以我現在反而不太想把它們排成單一名次。比較精確的說法應該是：

- `Codex` 比較像替長時間、多工具 agent workflow 做設計，`WebSocket` 與增量延續是它最鮮明的特徵
- `Claude Code` 在一些輔助任務上展現出不錯的成本意識，例如用小模型處理標題
- `Pi` 的輕量化是可觀測、可解釋的，不是空話，但它的省，也來自於功能面的克制

如果把這篇濃縮成一句話，那就是：我原本以為我是在比較四個工具的「速度」，最後其實比較到的是四種不同的產品取捨。這件事比單純的跑分更有意思，因為它更接近我們平常真正會遇到的問題：你到底要的是最完整的 agent 能力，還是更便宜、更輕、更可預期的互動成本
