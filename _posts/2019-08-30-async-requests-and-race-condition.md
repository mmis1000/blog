---
layout: post
title: 同步。非同步？失去同步！
---

今天你的程式 race condition 了嗎？

## 前言

因為 `非同步請求在意料外的時間點呼叫 callback` 弄壞程式 是人人都有過的經驗。

![程式是怎麼炸掉的？]({{ site.baseurl }}/assets/2019-08-30-async-requests-and-race-condition-1.jpg)

像上圖所見，你會簡單的因為在意料外的時間收到 callback 而導致程式 crash

以這段 code 作範例，它會因為 race condition 導致程式隨機出錯，拿到錯誤的結果

```js
let state = null

function request(url) {
    fetch(url).then(result => {
        console.log('latest result', result)
        state = result
    })
}

request('url1')
request('url2')
request('url3')
// 你的程式會出錯，最後得到的結果可能是其中任何一個而不是 url3
```

但我們有沒有可以讓這件事情不要發生，提早去避免他？而不是出錯再說？

## 方向

目前為止，有兩種做法

1. 讓在意料外的時間點觸發的 callback 完全沒有任何作用或絕對不會造成 crash（被動防禦）
2. 在離開想要接收 callback 的狀態之前主動清空所有 callback（主動防禦）

無論哪種都有有它的優缺點

## 被動防禦 (1)

像是一個作法，你可以用一個 tag 表達目前程式應該要接收被標上那些 tag 的 callback，
當 tag 改變時，之前的 callback 發現 tag 不同後直接無視後續流程

```js
let currentSession = Math.random()
let state = null

function withSession(fn) {
    var session = currentSession

    return function () {
        if (currentSession === session) {
            fn()
        } else {
            console.warn(new Error(`這個回調不該在現在發生`))
        }
    }
}

function request(url) {
    // kill previous callbacks
    currentSession = Math.random()

    var callback = withSession(result => {
        console.log('latest result', result)
        state = result
    })

    fetch(url).then(callback)
}

request('url1')
request('url2')
request('url3')
// ... 無論網路請求在任何順序下回來，我們的最終結果都是 url3
```

### 優點

1. 你只需要在很少的地方寫 guard 避免 callback 在奇怪的時機觸發
2. 可以簡單的處裡 `GET` 等安全操作

### 缺點

1. 發出請求的 action 事實上可能根本不希望副作用被直接取消掉（例如檔案上傳完成的彈窗通知之類）
2. 對於某些需要特殊處裡的非同步請求，  
   並不能被直接刪除 callback 就完事，  
   這樣可能導致網頁意外與伺服器失去同步，  
   像是當你刪除文章後，無論在任何狀況下，  
   本機端的文章紀錄都應該立刻清除，  
   不然會導致顯示出伺服器上已經消失的文章的狀況。

## 被動防禦 (2)

另一個做法則是確保程式的非同步請求之間不會互相覆蓋對方的資料，導致本機端出現預料外的狀態

```js
let state = {}

function request(url) {
    fetch(url).then(result => {
        console.log('latest result', result)
        state[url] = result
    })
}

request('url1')
request('url2')
request('url3')

/*
 * 無論請求的結束順序為何，最終結果都會是
 * {
 *     url1: ...
 *     url2: ...
 *     url3: ...
 * }
 */
```

對這個寫法而言，race condition 對它並不會造成任何問題，  
程式最終的狀態都一樣，適合處裡 request 可以安全的並行處理的狀況。

### 優點

1. 請求間的 race condition 問題本來就不會發生，因此也不需要特別處裡

### 缺點

1. 並不是所有程式都能改寫成這種對 race condition 安全的狀況，  
   有很多東西本質上就只能照順序執行，  
   例如傳送聊天訊息之類，  
   你不會希望到伺服器的聊天訊息的順序是完全隨機的，  
   因此你會希望上一個完全完成再進行下一個。  

## 主動防禦

在離開每一個狀態前清除 request

```js
let state = null
let cancelFunctions = []

function clearOldCallbacks () {
    cancelFunctions.forEach(cb => cb())
    cancelFunctions = []
}

function request(url) {
    var canceled = false
    var cancel = function () {
        canceled = true
    }

    fetch(url).then(result => {
        if (!canceled) {
            console.log('latest result', result)
            state = result
        }
    })

    cancelFunctions.push(cancel)
}

request('url1')

//...
clearOldCallbacks()
request('url2')

//...
clearOldCallbacks()
request('url3')
```

改由呼叫的人決定舊的 callback 是不是需要取消掉或是保留，  
api 只負責提供方法取消而不參與取消與否的邏輯

### 優點

1. 呼叫者可以決定每一個狀態改變時，callback 應該保留或著銷毀

### 缺點

1. 你需要在每一個狀態轉移時處裡掉每一種callback，不然程式就有機會崩潰。
2. 隨著狀態越來越多，狀態間改變的方式也會跟著增加，  
   直到某一個點後，手動處裡每一個狀態而不漏掉會變得幾乎不可行，  
   很難不靠 IDE 或 library 一個不漏的處裡掉所有種類 callback

## 結語

解決 race condition 的方式很多，  
但有時候卻不能簡單通用，按照情況決定適合的方法是必要的，  
在此同時，注意代碼中非同步請求的使用方式，  
也可以把大部分非同步問題提早避免，  
而不是最後才因為程式隨機發生錯誤煩惱。

Happy programming!
