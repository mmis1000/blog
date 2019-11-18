---
layout: post
title: 常見的 vue 模式，以及你什麼時候該用他們。
---

什麼時候你該用 computed，什麼時候該用 watch

## 前言

Vue 是一個可以根據你修改的資料自動更新畫面的 framework (又稱 reactive programing)， 在此同時 vue 又提供了各種工具，允許你去觀測資料變化 (watch)，或是轉換資料格式已呈現到 UI 上 (computed)，但是有時候，你可能會不知道用什麼方式才能最簡單的達成你的目的。

## 比較

以 data/computed/watch/method 為例

Q: 什麼東西能直接被模板使用？  
A: data/computed/method，在這三個東西上的屬性可以直接在模板中使用

Q: 什麼東西設計上不該造成副作用？  
A: computed，他本身不該造成任何寫入動作

Q: 設計上你可以修改 data 的時機是什麼？  
A: 觸發事件時、watch 到資料變更時、其他外部事件

### 如果以權責區分的話

Template 負責輸出頁面，  
Data 保存用來應該顯示到縣夜市的資料  
Computed 用來預先轉換 data，以使用在 template 上  
Watch 觀測資料變更，並且在資料變更後觸發特定動作  
Method 單純是個整裡 function 的地方，允許寫在他上面的 function 在整個 component的範圍內被使用  

## 如果以片語的方式表達的話

我們可以把 template / watch / computed 所做的事用特定句型表達

### 在(A)時顯示(B) (v-if/v-else-if/v-else)

```html
<template>
  <B v-if="A" />
</template>
```

#### 範例

當 `counter 變為 0` (A) 時顯示 `時間到` (B)

```html
<template>
  <div v-if="counter === 0">時間到</div>
</template>
```

### 顯示 (A) 經過 (B) 過程所得到的結果 (computed)

- 副作用：無

```html
<script>
export default {
  data: { A },
  methods: { B },
  computed: {
    result () {
      return this.B(this.A)
    }
  }
}
</script>
```

#### 範例

顯示 `counter` (A) `取整數的` (B) 結果

```html
<script>
export default {
  data: {
    // (A)
    counter: 1.5
  },
  methods: {
    // (B)
    floor(input) {
      return Math.floor(input)
    }
  },
  computed: {
    result () {
      return this.floor(this.counter)
    }
  }
}
</script>
```

顯示 `表單上所有的欄位` (A) `是否都正確填寫` (B)

```html
<script>
export default {
  data: {
    // (A)
    form: {
      A, B, C, ...
    }
  },
  methods: {
    // (B)
    checkForm(form) {
      for (let key of Object.keys(form)) {
        if (!check(form[key])) {
          return false
        }
      }

      return true
    }
  },
  computed: {
    formValid () {
      return this.checkForm(this.form)
    }
  }
}
</script>
```

顯示 `動物清單` (A) 中 `是否含有老鼠` (B)

```html
<script>
export default {
  data: {
    // (A)
    動物清單: ['老鼠', '貓', '馬']
  },
  methods: {
    // (B)
    含有老鼠(list)) {
      return this.動物清單.contains('老鼠')
    }
  },
  computed: {
    是否含有老鼠 () {
      return this.含有老鼠(this.動物清單)
    }
  }
}
</script>
```

### 當(A)改變時進行(B)  (watch)

- 副作用： __有__

```html
<script>
export default {
  data: { A },
  methods: { B },
  watch: {
    A () {
      this.B()
    }
  }
}
</script>
```

#### 範例

當 `counter 變為 0` (A) 時 `將用戶導到首頁` (B)

```html
<script>
export default {
  data: {
    counter: 9
  },
  methods: {
    // (B)
    goHome() {}
  },
  watch: {
    // (A)
    counter (newVal) {
      if (newVal === 0) {
        this.goHome()
      }
    }
  }
}
</script>
```

### 複合描述

這些句形可以互相組合，變成更複雜的句型，  
像是我們可以這樣組合上面的範例，  
只要你能表達的句子，就能有對應的程式碼。

#### 範例

```txt
在[[表單上所有的欄位][是否都正確填寫]]為非時，[顯示表單錯誤](v-if)
|-表單上所有的欄位是否都正確填寫(computed)
| |-表單上所有的欄位
| |-是否都正確填寫
|-顯示表單錯誤
```

```html
<template>
    <div v-if="!formValid">表單錯誤</div>
</template>
<script>
export default {
  data: {
    // (A)
    form: {
      A, B, C, ...
    }
  },
  methods: {
    // (B)
    checkForm(form) {
      for (let key of Object.keys(form)) {
        if (!check(form[key])) {
          return false
        }
      }

      return true
    }
  },
  computed: {
    formValid () {
      return this.checkForm(this.form)
    }
  }
}
</script>
```

## 關於 watch 的使用時機

A: __別用__

其實大多數情況下你不該用它，
除了特地造成不可逆變化的狀況（像是頁面挑轉之類），
watch 都只會讓你的程式更難除錯。

在 watch 中修改 data 的話，  
就等於是在資料更新後又立刻清掉舊的資料（因為你寫入了data，原本的data當然不見了），  
而且你無法找到修改來源（vue devtool 沒辦法告訴你最後一次修改data的是誰），  
在有超過一個以上的路徑觸發資料修改的情況下，  
除非使用 debugger，  
你將完全無法知道為什麼欄位被修改成你意料之中的數值。

如果 watch 中修改的 data 又觸發其他 watch 的話，  
這個情況將更加嚴重，  
你將會有更多的出錯路徑，  
更多的資料連鎖錯誤，  
而你完全不知道在什麼開始時機點出錯的（畢竟舊資料被覆蓋了）。

除非你有什麼用 `computed/data` 無法實現的問題（例如手動追蹤 parent 來的 property 來配合第三方 library 之類），你不應該用 `watch`，他是一把會刺傷你的手的雙面刃，你不該在沒有理由的情況下去用它。

## 我該怎麼決定用什麼

怎麼決定該將計算完的資料放進 data 或是直接在 computed 內計算出資料？

```txt
你要顯示的東西可以從其他欄位直接計算嗎？
        |
    --Y---N--
    |       |
computed   data
```

_如果你可以直接把它算出來，那你實在沒有理由去再另外存一份它_

## 結語

Vue 跟 jQuery 等 DOM 操作工具在設計理念上有所差異，  
不再是你拿著資料自己修改畫面，而是你定義了畫面如何依照規則改變後，由 Vue 幫你更新所有的畫面，  
有時候，需要更多的思索才能跳出舊的思考框架，貼合 Vue 設計理念的代碼，  
但是當你成功進入 vue 的思考模式後，你可以從中受益，  
寫出更接近你的思考的代碼，去掉不必要的中間過程（寫 `你想出現的效果` ，而不是寫 `達成你想出現的效果的步驟` ），  
這可以讓你寫出更簡短而且也更容易被人理解的代碼。

而這描述最終結果而不是過程的方式也就是 Declarative programming（宣告式程式設計），也是目前其他主流框架(react/angular)的設計。

會在你學習其他主流框架時給你很大的幫助。
