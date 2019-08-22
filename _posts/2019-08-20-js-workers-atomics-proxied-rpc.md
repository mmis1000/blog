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

#### TL;DR 我只對 code 有興趣

[Here you are, DOM Proxy(GitHub repository)](https://github.com/mmis1000/DOM-Proxy/tree/5a0ac8b7a331f694619413dd738e8cd9dadcc37b)

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
      - 你必須要能在 callback 裡呼叫 `preventDefault` 才有用啊！
3. 我想要直接在 web worker `document.body.appendChild(document.createElement('div'))` 可不可以？
   1. 如果用 `Proxy` 攔截全部性來模擬 DOM 物件可行嗎？
   2. 如何讓 worker 裡的 placeholder 能永遠指向 main thread 正確的 DOM 物件？
   3. 三等號怎麼辦？要怎麼避免不同 placeholder 指向同一個物件導致比較失敗？
   4. 垃圾回收怎麼辦？一直製造 placeholder 的話不會漏記憶體嗎？

## 用到的技術

1. Web Worker
   - Web 裡的 multi thread
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

### 1.1 Atomic.wait/Atomics.waitAsync

跨 worker 的呼叫機制

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

可是主頁面不給 Atomics.wait 啊？他對我們要做的事毫無疑問是必要的，有沒有 workaround？

[TC39 Issue](https://github.com/tc39/ecmascript_sharedmem/issues/100)

看起來，有人提到，ecmascript 內部是用 busy spining 來模擬 atomics.wait，這會動嗎？讓我們實驗一下

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

在原始的值跟預期不一樣時就不會去更改記憶體中的值，並且會返回原本的值，所以只要比較他回傳的值跟預期的原始值就知道變更成不成功

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
function handle (req) { /* ... */ }
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
function handleMessage(req) { /* whatever */ }

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
Blocking   Main thread         Worker thread   Blocking
   v           |
   v           | ----- request  ----> |           v
   v           |                    | |           v
   v           | | <-- request  --- | |           v
   v           | | --- response --> | |           v
   v           |                    | |           v
   v           | <---- response ----- |           v
   v           |
```

能行嗎？看起來能（然而我也不知道到底還有沒有哪裡寫錯 orz）  
在這個時間點，就已經有一個雙向同步 RPC 的雛形了  
可以對最開始的問題回答 **YES，的確可能**

不過我們能不能更進一步？

### 3.1 Proxy

請問我可不可以用 Proxy 直接 relay 全世界？  

看起來好像可以，proxy 有很多的 trap 可以修改一個動作的回傳結果，像是

```js
var newObject = new Proxy(oldObjectOrFunction, {
   set () {
      // 改變 newObject.xxx = ooo 的結果
   },
   get () {
      // 改變 var ooo = newObject.xxx 的結果
   },
   apply () {
      // 改變 newObject() 的結果
   },
   construct () {
      // 改變 new newObject() 的結果
   }
   //... 還有其他很多的
})
```

而為了達成我們接下來要做的事，我們至少要攔截 `get` , `apply`

讓我們試試

```js
var fun = new Proxy({}, {
   apply () {
      console.log('trapped')
   },
   get () {
      return 1
   }
})
fun()
console.log(fun.a)
```

我們得到了...錯誤訊息一枚🌚

```txt
Uncaught TypeError: fun is not a function
```

看來要 callable 需要原本的物件也是 function，  
讓我們再一次

```js
var fun = new Proxy(() => {}, {
   apply () {
      console.log('trapped')
   },
   get () {
      return 1
   }
})
fun()
console.log(fun.a)
```

大成功！ console 印出了我們寫的訊息，也顯示了改變回傳值後的屬性

### 3.2 讓 worker 裡的 placeholder 對應到 dom 物件吧

首先，我們只能傳遞可序列的資料過去 rpc 啊？不能直接傳遞物件的 reference  
那，弄個 id 來對應物件可不可行？

首先讓我們 main page 這邊弄個 map 對應 id 到物件

```js
// main page
/**
 * @type {Map<number, object>}
 */
var idToObjectMap = new Map()

// worker
// ...
```

讓我們把 main page 的 window 對應的 id 送到 worker

```js
// main page
/**
 * @type {Map<number, object>}
 */
var idToObjectMap = new Map()
var idOfWindow = getId()
idToObjectMap.set(id, window)

sendIdToWorker(idOfWindow)

// worker
// ...
```

然後在 worker 這邊接收 id，並用這個 id 產生一個 placeholder

```js
// main page
// ...

// worker
var idOfMainWindow = getIdFromMain()
var fakeWindow = new Proxy({}, {
   // .... use the idOfMainWindow somewhere
})
```

接下來 main window 這邊按照 worker 傳過來的 id 比較從 map 找到物件就好，之後重複一樣步驟（ apply 同理）

```js
// main page
// ...
var [requestObjectId, prop] = getGetRequestFromWorker()
var propertyValue = idToObjectMap.get(requestObjectId)[prop]

var idOfPropertyValue = getId()
idToObjectMap.set(idOfPropertyValue, window)

sendIdToWorker(idOfWindow)
// worker
// ...
var fakeWindow = new Proxy({}, {
   get (target, prop, receiver) {
      var id = getRemoteProperty(idOfMainWindow, prop)
      var object = createFakeObject(id)
      return object
   },
   apply (target, thisArg, argumentsList) {
      return callRemoteFunction(id, thisArg, argumentsList)
   }
})
console.log(fakeWindow.document)
```

實驗完畢，problem solved

### 3.3 讓物件比較正常運作

就如同前面展示的，  
每一次 get 呼叫 createFakeObject 產生的都是不同 wrapper 啊?

所以就會變成

```js
// worker
fakeWindow.document == fakeWindow.document // false???
```

這肯定不太對，而且這邊有兩個問題

1. main page 每次送來得對應 document 的 id 都不一樣，worker 沒辦法知道實際上是一樣的 object
2. worker 每次都會產生新的 wrapper，導致物件比較失敗

針對第一個問題，我們加一個 map 記憶之前有送到 worker 過的物件，如果下次需要送一樣的東西就用之前的 id

```js
// main page
// 送出物件到 worker 時
/** start */
var objectToIdMap = /** @type {WeakMap<object, number>} */ new WeakMap()
/** end */

var idOfWindow = getId()
idToObjectMap.set(id, window)

/** start */
objectToIdMap.set(window, id)
/** end */

// 存取屬性時

var [requestObjectId, prop] = getGetRequestFromWorker()
var propertyValue = idToObjectMap.get(requestObjectId)[prop]

/** start */
if (objectToIdMap.has(propertyValue)) {
  return sendIdToWorker(objectToIdMap.get(propertyValue))
}
/** end */

var idOfPropertyValue = getId()
idToObjectMap.set(idOfPropertyValue, window)

sendIdToWorker(idOfWindow)
```

問題一解決！接下來 worker 對一樣的物件一定會拿到一樣 id 了。

讓我們來解決問題二

我們之前每次拿到新的 id 都是直接產生新的 proxy，我們可不可以讓一樣的 id 拿到之前的 proxy？  
把 ID 跟 proxy 對應記起來如何？像是這樣的

```js
// worker
/** start */
var cachedIdToFakeObject = new Map()
/** end */

// ...
var fakeWindow = new Proxy({}, {
   get (target, prop, receiver) {
      var id = getRemoteProperty(idOfMainWindow, prop)
      /** start */
      if (cachedIdToFakeObject.has(id)) {
         return cachedIdToFakeObject.get(id)
      }
      /** end */

      var object = createFakeObject(id)

      /** start */
      cachedIdToFakeObject.set(id, object)
      /** end */

      return object
   },
   apply (target, thisArg, argumentsList) {
      return callRemoteFunction(id, thisArg, argumentsList)
   }
})
```

能行

```js
// worker
fakeWindow.document == fakeWindow.document // true 🎉
```

問題解決

### 3.4 搞定 GC

Q: 我們剛才...是不是完全沒請理掉 map 裡的東西過，這樣真的能行嗎？？？  
A: **毫無疑問，記憶體會漏**  
Q: 可是我要怎麼知道 map 裡的東西能清了？  
A: 好像年初 chrome 有個草案就是為了這個存在的  
Q: 哪個？  
A: [WeakRef (stage3) & FinalizationGroup (stage2)](https://github.com/tc39/proposal-weakrefs)

Let's try it.

首先，先把保存物件的 map 改成保存 WeakRef 以免導致物件無法回收

```js
      // ...
      // cachedIdToFakeObject.set(id, object)
      cachedIdToFakeObject.set(id, new WeakRef(object))
      // ...

```

然後利用草案中的 FinalizationGroup 追蹤物件到底能回收了沒

```js
var finalizationGroup = new FinalizationGroup(iter => {
   for (let id of iter) {
      clearUp(id)
   }
})

// 產生 fakeObject  的地方...
finalizationGroup.register(fakeObject, fakeObject的Id, fakeObject)
// ...
```

OK, 這樣問題就解決了，記憶體不漏了

## 總結

這次的實作中，把一些平常少用的 API 都試了一輪，也嘗試了幾個還在草案中的新 api，算是一次有趣的旅程，然後我把嘗試結果都放在這裡了，有興趣的去看看吧（或是鞭我寫出了什麼大 bug🌚🔫）

[DOM Proxy(GitHub repository)](https://github.com/mmis1000/DOM-Proxy/tree/5a0ac8b7a331f694619413dd738e8cd9dadcc37b)

## Reference

1. [Proxy](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
2. [Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
3. [Atomics.asyncAwait (stage2)](https://github.com/tc39/proposal-atomics-wait-async) (2019/08/20)
   - 有 polyfill
4. [WeakRef (stage3)](https://github.com/tc39/proposal-weakrefs) (2019/08/20)
   - only chrome 76+
5. [FinalizationGroup (stage2)](https://github.com/tc39/proposal-weakrefs) (2019/08/20)
   - only chrome 76+
