---
layout: post
title: 如何讓 Hinet 的 Youtube 不卡
---

Hinet 拜託你饒了我吧，比原始載點還慢的快取到底是搞甚麼飛機？

## 前提
1. 你是中華電信用戶
2. 你的頻寬方案明明應該夠順順看 Youtube 1080p 甚至 4K（12M 下載以上）
3. 你的 Youtube 卡到連看 720P 都有障礙
4. 在 `命令提示字元` 裡執行 `ping 8.8.8.8` 出現的 ping 大部分都不超過 50

# 恭喜你，你有可能成了中華電信 Youtube 快取節點的受害者

## 為什麼只有中華電信用戶會發生這種事？（像是台X大/台灣星星什麼的都沒事）
Well... Let me explain this for you.

1. 中華電信在他的內網內有一個 Youtube 的快取伺服器 [*注1](#備注)
2. 這個伺服器完全沒在維護， __非常爛__ ， **天殺的爛** ， <span style="font-size:3em">無敵爛</span>
3. 在他架設快取伺服器省了對外流量的錢後，就完全不管用戶死活，也不管用戶到底能不能正常看 Youtube
4. 因此，只要是中華電信的用戶，你開啟 Youtube 又剛好連上那個伺服器，那你就死定了，你的影片會卡到完全無法正常收看

## 要怎麼確定我是不是真的是中華電信受害者，<br>或著只是剛好我的 wifi 基地台什麼的爛掉？
1. 用電腦瀏覽器打開任何一個讓你覺得卡到受不了的 Youtube 影片
2. 按下鍵盤上的 F12
3. 切換到網路頁籤，會列出這個網頁所有的網路請求
4. 隨便找一條類似 `https://r5.sn-ipoxu-uool.googlevideo.com/videoplayback?....` 這樣的網址（有寫到 videoplayback 的）
5. 在 `命令提示字元` 中輸入 `tracert r5.sn-ipoxu-uool.googlevideo.com` (這裡的網址就是你剛剛看到的網址裡的伺服器名稱)
6. 你會得到類似這樣的結果 

```
在上限 30 個躍點上
追蹤 r5.sn-ipoxu-uool.googlevideo.com [202.39.143.80] 的路由:

  1    46 ms    38 ms     9 ms  192.168.1.1
  2    11 ms    13 ms    21 ms  h254.s98.ts.hinet.net [168.95.98.254]
  3    14 ms     9 ms     9 ms  tne1-3301.hinet.net [168.95.54.30]
  4    14 ms    22 ms    22 ms  tne1-3012.hinet.net [220.128.27.90]
  5    13 ms     9 ms    12 ms  tncd-3312.hinet.net [220.128.27.25]
  6    47 ms    19 ms    33 ms  202-39-143-80.hinet-ip.hinet.net [202.39.143.80]

追蹤完成。
```

7. 如果你發現最後一個 ip 的網域紀錄是像這個範例中的 `xxx.hinet-ip.hinet.net` 的話，恭喜你，你已經確定是中華電信 Youtube 快取節點的受害者了
8. 先把後面 [] 裡的 ip 記下來，待會會用到（範例中是 202.39.143.80）

### 如何處裡這個問題 (workaround)
1. 打開你電腦的防火牆的進階管理（每個windows版本位置可能不一樣，看網路找一下，或是執行 `mmc.exe wf.msc`）
2. 點輸出規則<br>![image](https://i.imgur.com/EfUQWx1.png)
3. 點新增規則<br>![image](https://i.imgur.com/1yvdzhb.png)
4. 點所有程式<br>![iamge](https://i.imgur.com/41xc3nP.png)
5. 通訊協定點 TCP<br>![iamge](https://imgur.com/cCV3Xvb.png)
6. 遠端 IP 位址選 這些IP位址 後按 新增<br>![iamge](https://i.imgur.com/U1pE7vV.png)
7. 貼上剛剛你 tracert 時，最後面的那個 IP<br>![iamge](https://i.imgur.com/YjUPyoG.png)
8. 選擇封鎖<br>![iamge](https://i.imgur.com/3mn6IFX.png)
9. 全勾<br>![iamge](https://i.imgur.com/PbvECpK.png)
10. 隨便寫個名稱或把你對中華的淦意都寫在名稱跟描述裡<br>![iamge](https://i.imgur.com/4brIFX4.png)
11. 按確定
12. 重新整理剛剛你覺得卡到爆炸的 Youtube 網頁，這時他應該會發現 hinet 裡的節點連不上，自己跳到其他的伺服器（第一次會久一點），如果不確定，可以看 f12 裡的 網路 頁籤確認伺服器是否真的改變了
13. 如果有一天中華修好了這個快取伺服器，不再會卡了，你只要把這條規則刪除就好了（估計沒戲）

### 如何處裡這個問題 (the true way)
1. 投訴中華電信直到他修好這個問題（中華電信：我會修好他我跟妳姓）

### 備注
1. Youtube快取伺服器，主要用以加速影片讀取並且節省 ISP 的對外流量，Google 在很多 ISP 的內網都有，這是一種很常見的方式，通常伺服器是由 ISP 架設
