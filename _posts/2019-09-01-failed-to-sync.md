---
layout: post
title: 同步。同步？失去同步！
---

今天你的程式跟 UI desync 了嗎？

## 前言

[接上集]({{ site.baseurl }}/2019-08-28-states-getters-and-bugs)

寫程式的過程中，你有沒有發生過， `按了表單忘記更新變數` ， `hash，變了存在變數裡的網址卻沒更新到` 等種種事故呢？

有沒有簡單的方式找到自己究竟漏掉了什麼？

要怎麼找到什麼變數需要更新？

## 開場

讓我們看看一段簡單的程式

```js
var a = 1
var b = 2
var c = 3

function aCrossBPlusC() {
    return a * b + c
}

console.log(`a * b + c = ${aCrossBPlusC()}`)
```

我們有一個簡單的函數能輸出 `a 乘 b 加 c`，而且他也只做這個簡單的功能。

![狀態依賴]({{ site.baseurl }}/assets/2019-09-01-failed-to-sync-1.png)

我們可以用一個簡單的圖表表示，我們怎麼得到答案的

我們把 a 乘上 b 後加 c

然而現實中，當你有幾百個變數時，這樣每次都從頭算太沒效率了，我們可能會想要把中間產物 cache 起來。

所以把這個程式完全用變數展開後是這樣的

```js
var a = 1
var b = 2
var c = 3

var aCrossB = a * b
var aCrossBPlusC = aCrossB + c

console.log(`a * b + c = ${aCrossBPlusC}`)
```

輸出看起來沒錯。

這個 pattern 很常見，通常也沒什麼問題．．． 直到你需要更新 `a`, `b`, `c`——

你開始遇到，為什麼 `aCrossB` 不對？我哪裡漏更新了 `aCrossBPlusC` ？種種問題。

```js
var a = 1
var b = 2
var c = 3

var aCrossB = a * b
var aCrossBPlusC = aCrossB + c

function setA(val) {
    a = val
    aCrossB = a * b
}
function setB(val) {
    b = val
    aCrossB = a * b
}
function setC(val) {
    c = val
    aCrossBPlusC = aCrossB + c
}
setC(1)
setB(2)
setA(3)

console.log(a, b, c, aCrossB, aCrossBPlusC)
// 為什麼 aCrossBPlusC 不對了？？？🤔
```

更糟糕的是，現實中的程式更大更複雜，這些需要更新的變數或許是十幾個甚至幾十個，你幾乎不可能一眼就看出哪裡就出錯了。

## 除錯

以上面的程式當範例，我們回到最開始的關係圖

![狀態依賴]({{ site.baseurl }}/assets/2019-09-01-failed-to-sync-1.png)

我們可以看到

`a` 往下走能找到 `aCrossB`, `aCrossBPlusC`  
`b` 往下走能找到 `aCrossB`, `aCrossBPlusC`  
`c` 往下走能找到 `aCrossBPlusC`

然而我們的程式裡，我們在修改 a, b, c 時實際上改了啥呢？

`setA` 改了 `aCrossB`，那 `aCrossBPlusC` 去哪了？  
`setB` 改了 `aCrossB`，那 `aCrossBPlusC` 去哪了？  
`setC` 改了 `aCrossBPlusC`

我們可以從剛才的關係圖找到，我們到底漏了什麼。

```js
var a = 1
var b = 2
var c = 3

var aCrossB = a * b
var aCrossBPlusC = aCrossB + c

function setA(val) {
    a = val
    aCrossB = a * b
    aCrossBPlusC = aCrossB + c
}

function setB(val) {
    b = val
    aCrossB = a * b
    aCrossBPlusC = aCrossB + c
}

function setC(val) {
    c = val
    aCrossBPlusC = aCrossB + c
}

setC(1)
setB(2)
setA(3)

console.log(a, b, c, aCrossB, aCrossBPlusC)
// 這次就對了
```

這個關係圖可能很簡單，可能很複雜，但最終都能一步步找到問題，而不是瞎猜。

## 改良

上面那段當中。我們的 `aCrossBPlusC = aCrossB + c` 跟 `aCrossB = a * b` 是不是重複了好多次？  
我們直接一個一個更新了所有相關的數字。

![更新邏輯]({{ site.baseurl }}/assets/2019-09-01-failed-to-sync-2.png)

而且隨著代碼增加，重複次數可能越來越多。

有沒有什麼改進方式？

我們或許可以再加幾個 function 來算中間值，而不是直接寫在 setA, setB, setC 裡

```js
var a = 1
var b = 2
var c = 3

var aCrossB = a * b
var aCrossBPlusC = aCrossB + c

function setA(val) {
    a = val
    updateACrossB()
}

function setB(val) {
    b = val
    updateACrossB()
}

function setC(val) {
    c = val
    updateACrossBPlusC()
}

function updateACrossB() {
    aCrossB = a * b
    updateACrossBPlusC()
}

function updateACrossBPlusC() {
    aCrossBPlusC = aCrossB + c
}

setC(1)
setB(2)
setA(3)

console.log(a, b, c, aCrossB, aCrossBPlusC)
```

我們把他改由直接相鄰的上一層更新下一層。

看起來又更短了一點，而且重複的地方也不見了

![更新邏輯]({{ site.baseurl }}/assets/2019-09-01-failed-to-sync-3.png)

## 再次改良

可是又有了另一個問題，  
雖然是 `aCrossB` 需要 `a` 跟 `b`，  
但更新邏輯卻是寫在 `setA`, `setB` 裡的，  
如果他們在不同檔案中分散在不同地方，  
在檔案多的時候會很難閱讀。

我們能不能從 `aCrossB` 發起，在觀察到 `a`, `b` 改變時去更新 `aCrossB`。  
而不是主動由 `a`, `b` 那邊更新 `aCrossB`？

在這邊，我們常聽到的 [觀察者模式](https://zh.wikipedia.org/zh-tw/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F) 就出現了。

我們可以在觀察到改變將要發生時，去修改依賴到他的欄位。

先弄個簡單的 event emitter

```js
function createSubject () {
    var subject = {
        observers: [],
        observe(observer) {
            subject.observers.push(observer)
        },
        notify () {
            for (let observer of subject.observers) {
                observer()
            }
        }
    }

    return subject
}


// 測試一下
var subject = createSubject ()

subject.observe(() => {
    console.log('something happened')
})

subject.observe(() => {
    console.log('I am also listening')
})

subject.notify()
```

我們可以把最開始的 code 改成

```js
function createSubject () {
    var subject = {
        observers: [],
        observe(observer) {
            subject.observers.push(observer)
        },
        notify () {
            for (let observer of subject.observers) {
                observer()
            }
        }
    }

    return subject
}

var a = 1
var b = 2
var c = 3

var aCrossB = a * b
var aCrossBPlusC = aCrossB + c

var subjectA = createSubject()
function setA(val) {
    a = val
    subjectA.notify()
}

var subjectB = createSubject()
function setB(val) {
    b = val
    subjectB.notify()
}

var subjectC = createSubject()
function setC(val) {
    c = val
    subjectC.notify()
}

var subjectACrossB = createSubject()
function updateACrossB() {
    aCrossB = a * b
    subjectACrossB.notify()
}
subjectA.observe(updateACrossB)
subjectB.observe(updateACrossB)

function updateACrossBPlusC() {
    aCrossBPlusC = aCrossB + c
}
subjectACrossB.observe(updateACrossBPlusC)
subjectC.observe(updateACrossBPlusC)

setC(1)
setB(2)
setA(3)

console.log(a, b, c, aCrossB, aCrossBPlusC)
```

你不再需要知道你改動數值時，應該順便更新什麼東西，
而是由依賴你的數值，主動去要求在東西變動時被通知。

程式邏輯上看起來也乾淨了很多，雖然不比最開始簡單，但也算容易閱讀。

而這上面這幾個方式你或許看著很眼熟，因為他們其實在各大 ui 框架中都常常出現。
像是 component 屬性傳遞等，都會常常用到。

## 結語

當你在處理資料時，你心裡應該要知道 `我的資料怎麼來的`, `什麼時候要順便更新`，  
而不是 `我現在要改變哪個變數`。

如果真的無法直接想像出來，心智圖跟紙筆會是你的好朋友。

設計模式是用來幫助你實現你的想法的工具而不是聖經。

理解資料之間的關聯，才是最關鍵的地方，也是你寫出不會出錯的程式的關鍵。

Happy programming!
