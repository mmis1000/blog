---
layout: post
title: Vue 實作正確的 redo 以及 undo
---

怎麼做出正確支援瀏覽器上下頁的 redo 以及 undo 功能

## 前提
在各種繪圖軟體以及網頁中，前進後退都是很常見的功能，
在一般的網頁多頁應用中，提供上一頁以及下一頁的功能時，表單的狀態是存在伺服器的，
瀏覽器只要送出網址，取回網頁就好，不需要去煩惱網頁的狀態，
但是單頁應用中，這些都需要由瀏覽器自己處裡。

## 用到的東西
1. vue
2. [vue-navigation](https://github.com/zack24q/vue-navigation)
3. vue-router

## 前提

假設有這麼一個 route，跟兩個頁面 `/page1`, `/pages2`

```js
const routes = [{
    path: '/page1',
    name: 'page1',
    component: Page1,
  },
  {
    path: '/page2',
    name: 'page2',
    component: Page2,
  },
];

const router = new VueRouter({
  routes
});
```

為了能知道現在跳轉的 route，  
是 Vue router 有紀錄的頁面還是沒有的 (是因為上下頁而觸發，還是用戶點了連結)，  
你會需要追蹤 scrollBehavior，  
因為沒開過的頁面當然不會有卷軸位置的紀錄

```js
const router = new VueRouter({
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      // newly opened page
    } else {
      // old page
    }
  }
});
```

但是這時候，還是有一個問題，你還是不知道用戶按了上一步或下一頁，以這個範例來說

```
/page1 -> /page2 - > [/page1] # 現在的狀況，你在 /page1
> 用戶上一頁
/page1 -> [/page2] - > /page1 # 現在的狀況，你在 /page2
```

這種時候，你知道 route 跳轉了，而且是跳轉到曾經開過的頁面 (因為 savedPosition 有紀錄)，  
那你能知道用戶按了上一頁或下一頁嗎？  
很遺憾，沒辦法，這種時候，無論上按上一頁，或是下一頁，都會是曾經開過的頁面。

因此，這種時候就是 vue-navigation 上場的時機了。

```js
export default {
  name: 'app',
  router,
  data() {
    return {
      mydata: 'something'
    };
  },
  mounted() {
    this.$navigation.on('back', () => {
      // 用戶按了上一頁
    });

    this.$navigation.on('forward', () => {
      // 用戶按了下一頁
    });
  }
}
```

我們可以做一個表格，歸納出幾種組合

vue-router 紀錄 \ 瀏覽方向 | 前進                     |後退
--------------------------|-------------------------|----------------------
無                        |1. 按了連結或是程式觸發的跳轉|2. 按了上一頁，但是沒紀錄
有                        |3. 按了下一頁              |4. 按了上一頁

為了正確處裡前進後退，我們需要處理 2, 3, 4 這三種情況

Q: 什麼時候會出現 1 的狀況？  
A: 當你點擊頁面上的連結，或是控制 vue-router 去跳轉頁面時，或是重新整理後，按了下一頁  
Q: 什麼時候會出現 2 的狀況？  
A: 當你重新整理網頁後，vue router 丟失了所有頁面狀態時，你又按了上一頁  
Q: 什麼時候會出現 3 的狀況？  
A: 當你按下下一頁時  
Q: 什麼時候會出現 4 的狀況？  
A: 你按了下一頁而且 vue-router 有紀錄  

## 實作
為了能做到後退，我們需要一個陣列來儲存每一次跳頁前 APP 的狀態

```js
const records = [];
```

然後我們先把前進後退的狀態整合在一起

```js
const records = [];
let isGoing = null;

const router = new VueRouter({
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      if (isGoing === 'back') {
        // 狀態 4
      } else if (isGoing === 'forward') {
        // 狀態 3
      }
    } else {
      if (isGoing === 'back') {
        // 狀態 2
      } else if (isGoing === 'forward') {
        // 狀態 1
      }
    }
  }
});

export default {
  name: 'app',
  router,
  data() {
    return {
      mydata: 'something'
    };
  },
  mounted() {
    this.$navigation.on('back', () => {
      isGoing = 'back';
    });

    this.$navigation.on('forward', () => {
      isGoing = 'forward';
    });
  }
}
```

然後在主動觸發跳頁時，我們須要把目前的狀態 snapshot 一份儲存起來，這樣我們待會按下上一頁時才能還原狀態

```js
export default {
  name: 'app',
  router,
  data() {
    return {
      mydata: 'something'
    };
  },
  mounted() {
    this.$navigation.on('back', () => {
      isGoing = 'back';
    });

    this.$navigation.on('forward', () => {
      isGoing = 'forward';
    });
  },
  methods: {
    submitPage1() {
      records.push(JSON.parse(JSON.stringify(this.mydata)));
      this.$router.push({
        path: '/page2'
      });
    }
  }
}
```

按下上一頁時我們需要還原之前狀態，並把目前狀態站存一份起來，因此我們再加個新陣列

```js
const records = [];
const nextRecords = [];
let isGoing = null;

const router = new VueRouter({
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      if (isGoing === 'back') {
        // 儲存目前狀態到新陣列的頭
        nextRecords.shift(JSON.parse(JSON.stringify(vm.mydata))
        // 還原之前狀態
        vm.mydata = JSON.parse(JSON.stringify(records.pop()));
      } else if (isGoing === 'forward') {
        // 狀態 3
      }
    } else {
      if (isGoing === 'back') {
        // 狀態 2
      } else if (isGoing === 'forward') {
        // 狀態 1
      }
    }
  }
});
```

按下下一頁時，如果我們知道下一頁是什麼，就把狀態還原

```js
const records = [];
const nextRecords = [];
let isGoing = null;

const router = new VueRouter({
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      if (isGoing === 'back') {
        // 儲存目前狀態到新陣列的頭
        nextRecords.unshift(JSON.parse(JSON.stringify(vm.mydata))
        // 還原之前狀態
        vm.mydata = records.pop();
      } else if (isGoing === 'forward') {
        records.push(JSON.parse(JSON.stringify(this.mydata)));
        // 還原之前狀態
        vm.mydata = JSON.parse(JSON.stringify(nextRecords.shift()));
      }
    } else {
      if (isGoing === 'back') {
        // 狀態 2
      } else if (isGoing === 'forward') {
        // 狀態 1
      }
    }
  }
});
```

最後一個問題，如果我們一直按上一頁，按到紀錄裡都沒東西，那怎麼辦？
因此，我們要再加一個 counter，來表示我們瀏覽到歷史紀錄哪個位置

```js
const records = [];
const nextRecords = [];
let currentHead = 0;

const router = new VueRouter({
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      if (isGoing === 'back') {
        if (records.length !== 0) {
          // 儲存目前狀態到新陣列的頭
          nextRecords.unshift(JSON.parse(JSON.stringify(vm.mydata))
          // 還原之前狀態
          vm.mydata = records.pop();
        } else {
          currentHead--;
        }
      } else if (isGoing === 'forward') {
        if (currentHead === 0) {
          records.push(JSON.parse(JSON.stringify(this.mydata)));
          // 還原之前狀態
          vm.mydata = JSON.parse(JSON.stringify(nextRecords.shift()));
        } else {
          currentHead++
        }
      }
    } else {
      if (isGoing === 'back') {
        if (records.length !== 0) {
          // 儲存目前狀態到新陣列的頭
          nextRecords.unshift(JSON.parse(JSON.stringify(vm.mydata))
          // 還原之前狀態
          vm.mydata = records.pop();
        } else {
          currentHead--;
        }
      } else if (isGoing === 'forward') {
        // 什麼都不做，因為大概是用戶按的
        // 狀態 1
      }
    }
  }
});

export default {
  name: 'app',
  router,
  data() {
    return {
      mydata: 'something'
    };
  },
  mounted() {
    this.$navigation.on('back', () => {
      isGoing = 'back';
    });

    this.$navigation.on('forward', () => {
      isGoing = 'forward';
    });
  },
  methods: {
    submitPage1() {
      // 清空目前瀏覽位置
      currentHead = 0;
      // 刪除所有下一頁的紀錄
      nextRecords.length = 0;
      // 儲存上一頁紀錄
      records.push(JSON.parse(JSON.stringify(this.mydata)));
      // 跳轉頁面
      this.$router.push({
        path: '/page2'
      });
    }
  }
}
```

這樣一來，無論按上下一頁，就都能正常運作了
