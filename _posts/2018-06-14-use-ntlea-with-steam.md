---
layout: post
title: 如何直接從 Steam 列表裡開啟非UTF8日文遊戲
---

也就是說把 ntleas(或是其他語系模擬軟體) 捷徑加進 steam 的方法

## 為什麼我想這樣做？
因為我想方便直接從 Steam Link 主介面開遊戲，
還要跳出回桌面開實在太麻煩了。

## 前題
1. 你要有 ntleas 本體 (或是其他語系模擬軟體)
2. 你的語系模擬軟體要可以做出快速開啟遊戲用的捷徑 (重要)
3. 你要有你要開的軟體 (廢話)

## 步驟
1. 總之先用語系模擬軟體做出可以直接用的捷徑
2. 確認用這個捷徑真的能正常開啟遊戲 (確定點他打開不會亂碼)
3. 打開 Steam 選新增非 steam 遊戲  
   ![image](https://i.imgur.com/4JU7ObK.png)
4. 然後選 `瀏覽` 去選你要開的那個日文遊戲本體
   ![image](https://i.imgur.com/dY1M3Dk.png)
5. 這時候這個捷徑應該還是不能用的，因為直接開會亂碼 (等於沒用 ntleas 直接點遊戲)
6. 然後去收藏庫對捷徑按右鍵選 `內容`  
   ![image](https://i.imgur.com/p83lV5D.png)
7. 對第一步時製作的捷徑按右鍵，選 `內容`  
   ![image](https://i.imgur.com/3DDSAxp.png)
8. 捷徑的 `目標` 應該會是類似 `C:\xxx\ntleas.exe "C:\ooo\xxx.exe"  "C932" "L1041" "FMS PGothic" "P4"` 這樣的東西  
   捷徑的 `開始位置` 會是類似 `C:\ooo` 這樣的東西
9. 先把這兩個東西記下來
10. 回到第六個步驟時打開的視窗  
    ![image](https://i.imgur.com/p8lKbfU.png)
11. 把 Steam 捷徑的 `目標` 改成第一步驟捷徑的 `目標` 那一排參數的 `第一項`，然後按下 `[enter]` 存檔，
    以舉的例子來說就是 `C:\xxx\ntleas.exe` ，而且要確保他有被 `""` 包起來，不然會出錯
12. 把 Steam 捷徑的 `開始位置` 改成跟第一步驟捷徑的 `開始位置` 一樣，然後按下 `[enter]` 存檔，
    以舉的例子來說就是 `C:\ooo\` ，而且要確保他有被 `""` 包起來，不然會出錯
13. 打開 Steam 捷徑的 `附加啟動選項`  
    ![image](https://i.imgur.com/5xlr00R.png)
14. 把第一步驟捷徑 `第二項以後` 的內容貼上去，以舉的例子來說就是 `"C:\ooo\xxx.exe"  "C932" "L1041" "FMS PGothic" "P4"` ，
    然後按下 `確定` 存檔，最開頭的 `C:\xxx\ntleas.exe` 要刪掉，不然會有問題
15. 現在你應該能用那個 Steam 遊戲庫捷徑直接開非UTF8的日文遊戲了， Congrats!
