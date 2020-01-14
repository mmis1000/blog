---
layout: post
title: 如何自架低延遲直播
---

延遲從何而來？延遲如何避免？

## 前言

現在網路上有很多資源讓你很簡單的就架設自己的直播系統，像是 [ffserver(deprecated)](https://trac.ffmpeg.org/wiki/ffserver)、[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)、[nginx-http-flv-module](https://github.com/winshining/nginx-http-flv-module)，然而，你會發現，在一般不特別設定狀況下你最少會有 7~8 秒以上的延遲，那麼問題來了，為什麼延遲這麼大？

## 名詞

1. Frame  
  所謂影格，也就是你看到的影片的每一個畫面
2. Key frame  
  又稱 I frame，影片解碼的基準點，自身就可以解碼為完整的畫面，除了 I frame 外，其他類型的影格都需要參照他才能解碼
3. GOP (group of picture)  
  兩個 I frame 間的片段集合，播放器解碼影片的最小單位

## 直撥的方式

首先從直播的方式開始，我們可以簡單把直撥分成兩種

1. 推流
    - 由伺服器主動推送影片片段到用戶瀏覽器上
    - 可以達到較低的整體延遲，因為瀏覽器並不需要去猜測影片片段是否存在
2. 拉流
    - 由用戶瀏覽器主動拉取影片片段

## 延遲從何而來

我們可以把直撥簡單拆分為四個部分

1. 用戶電腦處理影片並上傳到直撥伺服器
2. 直撥伺服器處理影片 encode (optional)，對拉流的協議而言，直撥伺服器需要在這裡把影片拆成片段。
3. 直撥伺服器將影片傳送到用戶的電腦
4. 用戶的電腦解碼並且播放影片

## 問題

### Key frame (關鍵幀)

事實上瀏覽器並沒有辦法從影片任何位置開始播放影片。  
除此之外，對於拉流的協議，伺服器並沒有辦法在 Key frame 以外的地方切割影片（hls 要求影片的第一影格地須盡可能是 I frame）  
過少的 keyframe 會導致片段大小暴增，  
除此之外，一定要遇到 Key frame 後，瀏覽器或是任何的播放器才能解出影像。

#### GOP cache

為了允許新加入的 client 立刻撥開始播放影片，有的媒體伺服器會從上一次遇到 keyframe 的地方開始播放。
這會導致觀眾相較於直撥主落後一段時間

```txt
Frames IPPPPPPPPI
           ^ 直撥者
Frames IPPPPPPPPI
       ^ 觀眾
```

而在不使用 `GOP cache` 的情況則是，媒體伺服器會等到直撥主送出下一個 keyframe 後，才開始推送影片給觀眾

```txt
一開始
Frames IPPPPPPPPI
           ^ 直撥者
Frames IPPPPPPPPI
       ???

下一個 keyframe 開始
Frames IPPPPPPPPI
                ^ 直撥者
Frames IPPPPPPPPI
                ^ 觀眾
```

這種情況下則會影響開始播放的啟動延遲

因此

> 過長的沒有 key frame 的片段需要盡可能避免

### Fragment size (片段大小) (hls/dash only)

對於拉流的協議而言，片段大小是由伺服器的，而且瀏覽器無法決定每個片段各自有多長，  
所以在伺服器完成下一個片段並允許瀏覽器下載前，瀏覽器怎樣都拿不到最新影像，  
過長的片段大小會導致瀏覽器有非常高的延遲。

例如：  
在 safari 上，safari 會 buffer hls 3 個片段的大小，因此 4 秒的片斷將導致高達 12 秒的播放延遲

> 在不嚴重影響品質/流量的前提下，片段越小越好。

### Buffer size（緩衝大小）

對於一般影片而言，為了避免播放卡頓，我們一般會希望有盡可能大一點的 buffer，  
然而對於即時影像而言，你並不會希望 buffer 大小那麼大。

> 對於串流來說，`Buffer ~= 額外延遲` ，所以你不會想要毫無意義的大的 buffer，  

在不會造成嚴重卡頓的前提下，buffer 反而是越小越好，
這跟一般播放器的預設並不一致，所以你會需要調整設定來壓縮 buffer，

### Missed fetch (下載失敗)

過早開始播放下一個片段的話，可能會用到目前不存在的片段，  
這對用戶而言會造成播放突然卡頓（因為影像跟聲音突然停住了），  
特別是對於拉流的協議影響更大，  
瀏覽器在失敗的拉取片斷後，將會不得不再花半秒到一秒（看網路狀況）的時間重試，以拿到最新片段，  
甚至在某些情況下，可能導致播放器發生錯誤並直接停止，  
而且重新開始播放後，延遲將會驟增，因為瀏覽器並不會主動跳過前面片段。

> 雖然 buffer 越小越好，你依然需要避免 buffer 過小導致播放到尚不存在的片段（尤其是在拉流協議上）。  

過小的 buffer 會影響播放品質。

## 處理方案

### Key frame 間隔

以 ffmpeg 當發送端時，可以透過

```txt
-force_key_frames 'expr:gte(t,n_forced*秒數)'
```

強制插入 keyframe，避免 keyframe 間間隔過大的問題，  
秒數必須是拉流協議分段長度的因數，  
以免造成無法切割片段在特定位置的問題。

除此之外，為了必免造成參照後面片段導致延遲開始串流的問題，你可能需要禁止向後參照 **（會影響畫質！）**

```txt
-flags +cgop
```

### 片段大小

理想狀況是越小越好，但是考量抓取問題，最小最好在 1(非常穩定的網路環境)~2(一般狀況) 秒以上。

如果用 `ffempg` 生成 hls 片段的話，可以以

```txt
-hls_time 秒數
```

如果是 `nginx-rtmp-module` / `nginx-http-flv-module` 的話，可以加上

```txt
hls_fragment 1s;
```

進行調整。

### 緩衝大小

妳可以透過 `<video>` 元素的 buffered 屬性知道目前你有多少 buffer 可以播放。

`<video>` 的 buffered 屬性是一個 [TimeRanges](https://developer.mozilla.org/en-US/docs/Web/API/TimeRanges) 物件。

see also: [HTMLMediaElement##buffered](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/buffered)

#### 取得 buffer 長度

你可以透過

```js
var available = videoElement.buffered.end(videoElement.buffered.length - 1)
```

取得目前你可以播放的最大長度。

#### 取得目前播放位置

你可以透過

```js
var current = videoElement.currentTime
```

取得目前播放位置。

#### 取得未播放 buffer 長度

將上方兩個相減

```js
var available = videoElement.buffered.end(videoElement.buffered.length - 1)
var current = videoElement.currentTime
var remaining = available - current
```

就是目前 buffer 大小（單位為秒）

#### 控制 buffer 大小

##### 直接跳轉

透過修改 `currentTime` 可以直接跳轉影片元素到特定位置。

例如

```js
var available = videoElement.buffered.end(videoElement.buffered.length - 1)
var current = videoElement.currentTime
var remaining = available - current

// 延遲超過 4 秒
if (remaining > 4) {
    videoElement.currentTime = available - 2
}
```

優點

- 立刻跳轉到最後位置
- 很簡單

缺點

- 聲音／畫面會突然跳動
- 實際上你不能真的跳到你指定的時間，跳轉本身就會造成延遲，所以你沒辦法真的追到很緊的播放位置

##### 修改播放速度

```js
var available = videoElement.buffered.end(videoElement.buffered.length - 1)
var current = videoElement.currentTime
var remaining = available - current

// 延遲超過 4 秒就稍微加快播放速度
if (remaining > 4) {
    videoElement.playbackRate = 1.05
}
// 如果延遲小於就降低播放速度
else if (remaining < 2) {
    videoElement.playbackRate = 1
}
```

優點：

- 不會有突然跳動
- 可以做到比較精確的控制 `( < 2s delay on http-flv / < 3s delay on HLS/DASH )`

缺點

- 需要一段時間才能跟上目前位置
- 很複雜
  - 實際上你不能單純用目前的 buffer 大小決定，因為 buffer 本來就是會上下浮動的，瀏覽器每次都是一次解好幾個 frame 而不是一個，完全按照目前剩餘大小決定會導致忽快忽慢
  - 你會需要統計一段時間 buffer 大小決定最佳 buffer 目標，並且按照網路狀態決定你要多激進的壓縮 buffer 大小

### 其他：hls.js, flv.js 的設定

一般來說 `hls.js`, `flv.js` 的設定都是針對播放品質處理的，並不會對延遲特化，因此會造成額外延遲。
你可以透過犧牲一些品質來減少延遲

**以下設定將會造成播放容易卡頓**

#### hls.js

```js
var hls = new Hls({
  // 不要 buffer 過多片段才開始播放，在拿到最初片段後就直接開始
  liveSyncDurationCount: 1
})
```

#### flv.js

```js
const flvPlayer = this.player = Flv.createPlayer({
  type: 'flv',
  // 他是直播
  isLive: true,
  // 你的網址
  url: this.src,
  // 關掉緩衝 buffer
  enableStashBuffer: false,
  // 縮小最初播放需要的 buffer 大小
  stashInitialSize: 128,
  // 利用 webworker 增加解碼速度（實驗性）
  enableWorker: true
})
```

## 範例

### 伺服器端：nginx

https://github.com/mmis1000/http-flv/blob/bef023f4d30e4b08326a1054d8a7a9bbd7fdfe12/nginx/conf.d/rtmp/rtmp.conf

### 觀眾端：flv.js

https://github.com/mmis1000/http-flv/blob/6d8500b5743d5d4aa11859ed2f0c4256448cc2e1/web/src/components/FlvJs.vue

### 整合好的範例 docker

```shell
docker run --rm -it -p 1980:80 -p 1935:1935 mmis1000/nginx-http-flv:dev
```

網頁在 http://127.0.0.1:1980/

影片上傳網址為  rtmp://127.0.0.1:1935/demo/stream-1

### 直播主端：產生測試串流的 ffmpeg 指令

```shell
ffmpeg \
  -re -fflags +genpts \
  -stream_loop -1 \
  -i '你的影片' \
  -c:v libx264 -preset veryfast -maxrate 3000k -bufsize 6000k -pix_fmt yuv420p -g 50 \
  -c:a aac -b:a 160k -ac 2 -ar 44100 \
  -f flv \
  -force_key_frames 'expr:gte(t,n_forced*1)' -flags +cgop \
  -vf drawtext=fontfile=roboto.ttf:text='%{localtime}':fontsize=40:fontcolor=white@0.8:x=250:y=200 \
  '串流網址'
```

## 結論

雖然不像專有協議一樣可以達到小於一秒的極限延遲，  
只要用對方法，用開源方案做到低延遲還是做得到的，  
只是你會需要決定 `你要犧牲多少畫質`、`佔用多少流量`、`佔用多少轉碼資源` 來決定你想做到多少延遲。

而這個代價會需要你自己決定。
