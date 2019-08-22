---
layout: post
title: 為什麼你應該關閉 express.js 的延伸格式 query string
---

要怎麼（不）讓你的 express Hello World app 噴錯

## 讓我們從一個最基礎的 express app 開始

```ts
import * as express from "express";

const app = express();

app.get("/echo", (req, res) => {
    res.end(req.query.data);
});

app.listen(3000, () =>
    console.log("Server is running on http://localhost:3000")
);
```

## 問題

Q: 什麼都不做就只是吐出參數中的 `data` 而以，應該沒問題對吧？  
A: **他會因為用戶亂輸入參數吐出 500**

不相信？

試試看這個

```txt
http://你的網域:3000/echo?data[a]=1
```

你會得到

```txt
TypeError [ERR_INVALID_ARG_TYPE]: The "chunk" argument must be one of type string or Buffer. Received type object
```

But why?

Express.js 預設使用了類似 PHP 的延伸 query string 格式  
見 [qs](https://www.npmjs.com/package/qs)  
因此上面的網址中的 query string 會被轉換為

```json
{
    "data": {
        "a": "1"
    }
}
```

而試圖把物件往 `res.end` 塞，當然就噴出錯誤了

## 嘗試修復

Q: 那我們先用 String() 把參數轉 string 是不是就安全了？  
A: Well...yes, but actually no. 還是會爆炸  

不信？  

試試看這個

```ts
import * as express from "express";

const app = express();

app.get("/echo", (req, res) => {
    res.end(String(req.query.data));
});

app.listen(3000, () =>
    console.log("Server is running on http://localhost:3000")
);
```

```txt
http://你的網域:3000/echo?data[toString]=1
```

這次你得到了

```txt
TypeError: Cannot convert object to primitive value
```

但這又是為什麼？

這是因為，當 js 中要把物件轉為 string 時，他會呼叫物件上的 `toString` 方法，
然而要是你把 `toString` 屬性覆蓋掉的話，他就炸開了。

## 正確的修復方式

### Part 1

驗證輸入類型，如果不對直接吐出錯誤

```ts
app.get("/echo", (req, res) => {
    if (typeof req.query.data) !== 'string') {
        res.status(400).end('what are you doing???')
    }

    res.end(req.query.data);
});
```

### Part 2

關閉 express 的延伸 query string 格式以免以後一樣被亂喂參數  
這樣 express 就會改用 nodejs 內建的只支援簡單格式的 `querystring` 而不是 `qs`

```ts
app.set('query parser', 'simple')
```

這樣一來，問題終於解決了
