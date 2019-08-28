---
layout: post
title: 你可能不需要那麼多變數
---

探討 bug 的真正起因

## 前言

我們在寫程式時，或多或少都遇到 bug 過，

我們把遇到的 bug 給 patch 起來，寫個測試，
認為這就是修好了。

然後之後遇到新的bug，再次修復，再次寫測試。

不斷往復，重複這個輪迴，

提出了更多的design pattern，用了更複雜的測試方法。

然而，修得更多，生得更多。

這究竟是為什麼讓我們一直在這個輪迴受難？

## 為什麼

讓我們從最簡單的一個程式開始

```js
var a = false
var b = false

function setA(value) {
    a = value
}

function setB(value) {
    b = value
}

function print() {
    console.log(`a is ${a}, b is ${b}`)
}
```

我們有兩個 `setter` 分別設定 `a`, `b` 變數，以及一個 `print` function 輸出 `a`, `b` 目前的值。

簡單的試一下

```js
setA(true)
setB(false)
print()

// 輸出: `a is true, b is false`
```

看起來一切正常

目前這個程式有四種狀態

|a    |b    |
|-----|-----|
|false|false|
|true |false|
|false|true |
|true |true |

這些狀態都對嗎？答案是都對，畢竟目前就只是單純輸出而已

接下來我們加一個變數表示表示 a xor b，  
然後在設定完 b 時更新他

```js
var a = false
var b = false
var aXorB = false

function setA(value) {
    a = value
}

function setB(value) {
    b = value
    aXorB = a ^ b
}

function print() {
    console.log(`a is ${a}, b is ${b}, a xor b is ${aXorB}`)
}
```

再試一下

```js
setA(true)
setB(false)
print()

// 輸出: `a is true, b is false, a xor b is true`
```

看起來輸出是對的。

然而眼尖的你應該也發現哪裡不對勁了

要是我們改完 b 後再改 a 呢？

```js
setB(false)
setA(true)
print()

// 輸出: `a is true, b is false, a xor b is false`
```

歐不，程式出 bug 了

接下來我們再把 setA 也 patch 一下

```js
var a = false
var b = false
var aXorB = false

function setA(value) {
    a = value
    aXorB = a ^ b
}

function setB(value) {
    b = value
    aXorB = a ^ b
}

function print() {
    console.log(`a is ${a}, b is ${b}, a xor b is ${aXorB}`)
}
```

然後再寫一堆測試確定全部順序組合都沒錯

```js
it('should work when setA then setB', function () {
    setA(true)
    setB(false)
})
it('should work when setA then setB', function () {
    setB(false)
    setA(true)
})
// 其他高達幾十種組組合...
```

這些聽起來某種意義上很愚蠢對吧？

**然而我們差不多每天都在做，雖然不會這麼明顯，但事情都差不多。**

是不是哪裡跑偏了？有沒有更恰當的做法？

## 原因

讓我們回到源頭，什麼是 bug？bug 就是程式變成了不該出現的狀態，我們再做一次表格來看我們現在的程式所有的狀態組合

|a    |b    |AXorB|是bug嗎？|
|-----|-----|-----|---------|
|false|false|false|         |
|true |false|false|bug*     |
|false|true |false|bug      |
|true |true |false|         |
|false|false|true |bug      |
|true |false|true |         |
|false|true |true |         |
|true |true |true |bug      |

我們剛剛炸掉的程式在第二個組合，而這個是不該出現的組合。

而所謂的除錯就是避免程式意外進到錯誤的狀態組合中。

組合數量隨著狀態的變多，呈現 **級數增長**，直到某個時間點後，
就再也沒有任何一個人能把哪些狀態是該出現的，哪些是不該出現的給搞清楚。
只能出了 bug 後把出錯的路徑封死、寫個測試確保以後一定不會再出現。

然而問題的核心並不在這裡。

而是， **為甚麼我們的程式要能表達出 bug 的狀態？**

我們最開始的程式是不會有 bug 的，因為每一個狀態都是正確的，不存在能表達程式遇到 bug 的狀態。

## 為什麼加了一個新 field 後就爛了

因為你一開始就不該為一個完全依賴其他變數的東西加 field。

## 那怎麼辦

**請改用 getter**

讓我們用 getter 改寫剛剛出 bug 的程式

```js
var a = false
var b = false
// var aXorB = false

function setA(value) {
    a = value
}

function setB(value) {
    b = value
    // aXorB = a ^ b
}

function getAXorB() {
    return a ^ b
}

function print() {
    console.log(`a is ${a}, b is ${b}, a xor b is ${getAXorB()}`)
}
```

這個程式有 bug 嗎？
答案是 **不可能有**，因為它並沒辦法表達出 bug 的狀態。  
它的狀態表格跟最開始一模一樣，所有組合都是正確的，所以它 **不能** 出bug。  
你不可能得到 getAXorB() 不等於 a xor b 的結果。

也就是說：**getter 是不會增加額外的程式狀態的**

不像加一個field後更新時可能因為順序出問題，
如果 getter 的轉換規則是正確的，那它永遠對應到正確的數值，不可能意外出錯。
而更少的狀態組合也代表你的程式測試起來會更簡單，更難以出錯

## 結語

與其寫出 `測試完沒出 bug 的程式`，  
我想，寫出 `不能出 bug 的程式` 或許才是我們真正該做的。

monkey patch 修正每一個 bug 真的不太對，而且很可能只在專案還小時行得通，
隨著專案規模上升，要 patch 掉每條錯誤的流程難度將會暴增。
10 個 `boolean` 就是 1024 種狀態，10個 `int` 或許就能達到恆河沙數，
暴漲的狀態數量最終都將會把程式帶入難以測試除錯的陷阱。

只有真正思索每一個變數/屬性是不是必要，是否完全依賴什麼數值可以用 getter 替代，思考狀態組合的正確與否，才能到達沒有 bug的世界。
