---
layout: post
title: Vue SPA 完全 debug 攻略
---

一步一步快速有效 debug vue app

## 前提

電腦上需要安裝的軟體

1. FireFox or Chrome
2. Vue dev tool
    - FireFox: [Firefox Addon](https://addons.mozilla.org/zh-TW/firefox/addon/vue-js-devtools/)
    - Chrome:  [Chrome Addon](https://chrome.google.com/webstore/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd)
3. 一個適合的文字編輯器
    - 推薦 VsCode

## Step By Step

1. 在 Vue 的開發模式下打開出錯的頁面
    - 首先翻到發生錯誤的頁面
2. 打開開發者工具
    - Window: `F12`
    - Mac: `Option + Command + I`
3. 切到 Vue 的分頁
4. 使用 VDom 檢視工具選取出錯的元素
5. 找到正確的頁面
    - 如果是 SPA，檢查 `$route` 中的 route 資訊，並從 `route.js`（或依專案可能是 `router.js`）中找到正確的頁面 component
    - 如果不是，一般為 `src/App.vue`
6. 開你的編輯器，找到該元素在頁面中對應的代碼
7. 找到顯示在該元素的資訊來源（可能在 `computed` 或著 `data` 中）
8. 在 vue 的開發者工具找到對應的數值
9. 檢查數據
    - 如果是 data，檢查 data 跟預想中的資訊是否一致
        - 並找到修改這個 data 的函數，檢查過程發生了什麼問題
    - 如果是 computed，先檢查 computed 用到的所有資訊是否正確
        - 如果依賴的 store 裡的資訊，切換到 store 分頁查看 store 中資訊是否階正確
        - 如果 computed 依賴的資訊皆正確，則檢查 computed 本身計算過程是否錯誤
        - 如果依賴了另一個 computed，則遞迴以上過程，直到資料來源都確認無誤
10. 修正計算過程

## 結語

善用開發者工具可以幫助你更快的修正問題，讓你能在有限時間內找到錯誤的正確位置。
一步步往回追的方式能有效快速縮減問題可能來源，而不是在茫茫 code 海中找兇手。
