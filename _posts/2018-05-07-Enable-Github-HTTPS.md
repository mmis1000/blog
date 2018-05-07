---
layout: post
title: 如何開啟 GitHub pages 的強制 HTTPS 模式
---

Let us make your GitHub pages secure again!

## 前提
1. 你要有一個設定好 GitHub pages 的 repo
2. 你的 GitHub page 必須已經 [綁定了自定的網域](https://help.github.com/articles/quick-start-setting-up-a-custom-domain/)
3. 你已經設定過了 DNS 紀錄將網域指向 GitHub 的主機
4. 如果你使用了 `CloudFlare` 或 `CloudFront` 等 CDN 服務，你已經將 CDN 模式關閉  
  - 否則 GitHub 將沒辦法正確偵測到你已經將 DNS 紀錄指向正確的主機

## 步驟
1. 如果你原本的網域是用 `A 紀錄` ，跳到 `步驟 2` ，不然跳到 `步驟 6`
2. 移除原本的 `A 紀錄` ，把紀錄換到新的支援 HTTPS 的伺服器 [官方說明](https://help.github.com/articles/setting-up-an-apex-domain/#configuring-a-records-with-your-dns-provider)
3. 把 DNS record 放到過期，大概要半個小時到一個小時  
  - 這時候用 `dig +noall +answer yourdomain.com` 查詢你的ip，應該會顯示開頭為 `185.199.*.*` 的 4 個 ip 之一
  - 因為 GitHub 那端的 DNS 快取較久，建議多等幾分鐘（至少等 15 至 30 分鐘不等）
4. 移除 repo 內的 `CNAME` 檔案  
  - ![刪除 CNAME](https://i.imgur.com/fqkuXQU.png)
5. 將 `CNAME` 檔加回 repo (或是直接用 force push 回退到上一個 commit)
6. 這時候設定裡應該會寫說，因為 cert 還沒簽好，所以不能開啟強制 HTTPS，如果還是寫因為 DNS 設定問題所以不支援，回到 `步驟 3`  
  - 應該要是這樣 ![Cert 還沒簽好](https://i.imgur.com/CNKfVwe.png)
7. 等 (最長兩天) cert 簽好，如果遲遲沒有更新，那麼回到 `步驟 4`，否則進到 `步驟 8`
8. 去 repo 設定裡打開強制 HTTPS  
  - ![設定](https://i.imgur.com/TyGATnK.png)
  - ![強制 HTTPS](https://i.imgur.com/w3BRSdQ.png)
9. 確認 repo 裡沒用到 http 的資源導致網站炸裂，如果炸了就修好他
10. 完成了！😀
