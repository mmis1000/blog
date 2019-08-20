---
layout: post
title: 從 Atomics/Proxy 開始的跨 worker js RPC 之旅
---

Let's make it just works.  
讓我們直接在 web worker 讀寫 DOM 吧！ (like a boss)

## 前提(2019/08/20)

1. 目前並沒辦法直接在 web worker 中存取 DOM (因為她們活在不同世界中)
2. 目前所有的 js RPC 都是依賴事件機制
   - a.k.a. `postMessage` / `self.onmessage`
3. 目前所有存在的 RPC 都是非同步的
   - `postMessage` / `selfMessage` 永遠都會在下一個 tick 才會回來
4. 目前所有存在 RPC 都需要在外面包一層 RPC 本身的 API 來使用
   - 基於上方兩個理由，而且你並沒辦法讓非同步的簡單地變成同步的代碼

## 動機與疑問

### 動機

#### **到底為什麼要這樣做啊？ 🤔**

在幾個月前，某個地方的聊天室裡，聊天中，有人提出了一個疑問：「 `我可以在 web worker 中處理 dom 上的事件並且按情況 preventDefault() 嗎？` 」

看似很簡單的問題...然而卻是意料外的困難。
雖然目前跨 worker 的 rpc framework 很多，卻沒有任何一套能做到。

為什麼呢？

因為，`preventDefault()` 只有在 event callback 裡同步呼叫才有意義。
例如說，當你點了一個 &lt;a&gt; 連結，你在事件中呼叫 `preventDefault()` 可以阻止點下連結這個動作打開網頁。
這也代表，你必須要在網頁打開之前呼叫 `preventDefault()` 才能阻止打開網頁，
如果你在網頁打開後才 `preventDefault()`，那網頁已經打開了，當然半點功用都沒有。

而且在取消之後，你也不能手動觸發原本用戶觸發事件會引發的原生行為。  
就算你重新手動觸發click事件，你也不會打開網頁。
所以判斷完成後在事件裡同步呼叫 `preventDefault` 幾乎是唯一一條可行的路。

*然而目前沒有內建的簡單方式同步呼叫 web worker 處理事件。*

那...我們可以怎麼做？

除了 `Atomics.wait` 外，目前不存在任何 api 能在 worker 間同步傳遞事件  
（雖然他相當的低階，不容易使用，幾乎是系統記憶體操作的一對一對應。）  
但畢竟他是唯一一個可行的路，值得我們嘗試一番，所以就有了這篇文章

### 疑問

#### **我可以拿他做到些什麼？ 🤔**

1. 我有沒有可能用它做到直接從 web worker 讀寫 dom？
   1. 靠 `Atomics.wait` 同步呼叫真的可行嗎？
      - 可是呼叫 `Atomic.wait` 下去，worker 就當場 hang 了啊？
   2. 主頁面好像不給 `Atomics.wait` 耶？
   3. 要是兩邊同時呼叫導致 dead lock 死在那邊怎麼辦?
2. 我能不能靠它直接雙向讀寫 DOM？
   1. 在 `worker 呼叫 dom 並等待結果` 時，如果 `dom 又想呼叫 worker` 怎麼辦？
3. 我想要直接在 web worker `document.body.appendChild(document.createElement('div'))` 可不可以？
   2. 如果用 `Proxy` 攔截全部性來模擬 DOM 物件可行嗎？
   3. 垃圾回收怎麼辦？這樣不會漏記憶體嗎？

## 用到的技術

1. Web Worker
   - Web 裡的 multithread
   - [在 MDN 上的簡介](https://developer.mozilla.org/zh-TW/docs/Web/API/Web_Workers_API/Using_web_workers)
2. SharedArrayBuffer
   - 在不同 Web Worker 直接共享記憶體的 API
   - [在 MDN 上的 API 說明(英文)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
3. Atomics
   - 包含用以安全讀寫共享記憶體/等待資料變更等api
   - [TC39 的英文簡介](https://github.com/tc39/ecmascript_sharedmem/blob/master/TUTORIAL.md)
   - [在 MDN 上的 API 說明(英文)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
   - [Stack overflow](https://stackoverflow.com/questions/48346490/what-does-the-atomics-object-do-in-javascript)
   - Atomics.awaitAsync (stage2)
4. Proxy
   - 可以攔截事件呼叫，屬性存取等
   - [MDN](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
5. Weak reference (stage3)
   - 弱引用
   - [TC39 草案](https://github.com/tc39/proposal-weakrefs)
6. FinalizationGroup (stage2)
   - 追蹤記憶體回收的機制
   - [TC39 草案](https://github.com/tc39/proposal-weakrefs)

## 實驗

### 1.1 Atomics.waitAsync

測了一下 `Atomics.wait` ，一切正常

可是要是 wait 下去 thread 就死在那邊完全不動了啊 ！？

我能不能非同步呼叫 wait 以免卡死 event loop？

搜尋了一下，看來的確有個 waitAsync 草案希望可以在 non blocking 的情況下，去等待 sharedArrayBuffer 的資料變更，而且還有附上 polyfill，用法大概是這樣的

```js
var sab = new SharedArrayBuffer(1024)
var ia32 = new Int32Array(sab)

async function run () {
   var result = await Atomics.waitAsync(ia32, offset, originalValue)
}
```

而且主頁面也可以用，實驗成功

### 1.2 Busy spin wait

可是主頁面不給 Atomics.wait 啊？他對我們要做的事好無疑問是必要的，有沒有 workaround？

[TC39 Issue](https://github.com/tc39/ecmascript_sharedmem/issues/100)

看起來，有人提到，ecmascript 內部適用busy spining 來模擬 atomics.wait，這會動嗎？讓我們實驗一下

```js
// main page
var ia32 = getItSomeWhere()

async function wait (ia32, offset, originalValue, timeout = Infinity) {
   var start = Date.now()
   while (originalValue !== Atomics.load(ia32, offset)) {
      if (Date.now() - start > timeout) {
         return 'timed-out'
      }
   }

   return 'ok'
}

wait(ia32, offset, originalValue)

// worker
var ia32 = getItSomeWhere()
ia32[offset] = newValue
```

...看來能行，實驗成功

### 1.3 compareExchange

要怎麼避免兩邊同時試圖送給對方信息，有沒有辦法保證一定只有一邊成功先送訊息？

看了一下，有一個叫做 [Atomics.compareExchange](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange) 好像能保證只會有其中一邊變更記憶體，不會鬼打牆避免兩邊都認為自己成功

他接收四個參數，分別是 Int32Array, 偏移, 原始值, 新的值

在原始的值跟預期不一樣時就不會去更改記憶體中的值，並且會返回原本的值，所以只要比較他回傳的植根預期的原始值就知道變更成不成功

```js
// main page
var ia32 = getItSomeWhere()
var actualOriginalValue = Atomics(ia32, offset, oldValue, newValue)

if (actualOriginalValue === oldValue) {
   console.log('變更成功')
} else {
   console.log('變更失敗')
}

// worker
var ia32 = getItSomeWhere()
var actualOriginalValue = Atomics(ia32, offset, oldValue, newValue)

if (actualOriginalValue === oldValue) {
   console.log('變更成功')
} else {
   console.log('變更失敗')
}
```

問題解決

### 2.1 處理雙向呼叫

簡化過的代碼

```js
// Main
function sendMessage () {
   requestWorker(req)
   var result = pollResultFromWorker()
   return result
}

// Worker
function handleMessage(req) {
   const res = handle(req) // whatever
   responseToWorker(res)
}
```

考慮以下順序

```txt
Blocking   Main thread           Worker thread   Blocking
   v           |
   v           | ----- request  ----->  |           v
   v           | <---- response ----->  |           v
   v           |
```

在 main thread 送出 request 後，他會預期 worker 應該會回覆給他 response，這期間整個 main thread 是 blocking 的，
如果其他這時 worker 又想請求 main thread 怎麼辦？

如果...在收到回覆的時候加一個事件類型判斷是不是請求可不可行？

```js
// Main
function handleMessage(req) { /* whatever */}

function sendMessage () {
   requestWorker(req)
   while (true) {
      var result = pollResultFromWorker()

      if (isResult(result)) {
         break
      } else if (isRequest(result)) {
         handleMessage(result)
      }
   }
   return result
}

// Worker
function handle (req) {
   // ....
   requestWorkerMain(req + 1)
   // ....
}

function handleMessage(req) {
   const res = handle(req) // whatever
   responseToWorker(res)
}
```

順序變成

```txt
Blocking   Main thread           Worker thread   Blocking
   v           |
   v           | ----- request  ------> |           v
   v           |                      | |           v
   v           | | <-- request  ----- | |           v
   v           | | --- response ----> | |           v
   v           |                      | |           v
   v           | <---- response-------- |           v
   v           |
```

能行嗎？看起來能（然而我也不知道到底還有沒有哪裡寫錯 orz）

## Reference

1. [Proxy]((https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Proxy))
2. [Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
3. [Atomics.asyncAwait (stage2)](https://github.com/tc39/proposal-atomics-wait-async) (2019/08/20)
   - 有 polyfill
4. [WeakRef (stage3)](https://github.com/tc39/proposal-weakrefs) (2019/08/20)
   - only chrome 76+
5. [FinalizationGroup (stage2)](https://github.com/tc39/proposal-weakrefs) (2019/08/20)
   - only chrome 76+
