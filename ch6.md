# 第 6 章: 示例應用

## 宣告式程式碼

我們要開始轉變觀念了，從本章開始，我們將不再指示計算機如何工作，而是指出我們明確希望得到的結果。我敢保證，這種做法與那種需要時刻關心所有細節的指令式程式設計相比，會讓你輕鬆許多。

與命令式不同，宣告式意味著我們要寫表示式，而不是一步一步的指示。

以 SQL 為例，它就沒有“先做這個，再做那個”的命令，有的只是一個指明我們想要從資料庫取什麼資料的表示式。至於如何取資料則是由它自己決定的。以後資料庫升級也好，SQL 引擎優化也好，根本不需要更改查詢語句。這是因為，有多種方式解析一個表示式並得到相同的結果。

對包括我在內的一些人來說，一開始是不太容易理解“宣告式”這個概念的；所以讓我們寫幾個例子找找感覺。

```js
// 命令式
var makes = [];
for (i = 0; i < cars.length; i++) {
  makes.push(cars[i].make);
}


// 宣告式
var makes = cars.map(function(car){ return car.make; });
```

命令式的迴圈要求你必須先例項化一個數組，而且執行完這個例項化語句之後，直譯器才繼續執行後面的程式碼。然後再直接迭代 `cars` 列表，手動增加計數器，把各種零零散散的東西都展示出來...實在是直白得有些露骨。

使用 `map` 的版本是一個表示式，它對執行順序沒有要求。而且，`map` 函式如何進行迭代，返回的陣列如何收集，都有很大的自由度。它指明的是`做什麼`，不是`怎麼做`。因此，它是正兒八經的宣告式程式碼。

除了更加清晰和簡潔之外，`map` 函式還可以進一步優化，這麼一來我們寶貴的應用程式碼就無須改動了。

至於那些說“雖然如此，但使用命令式迴圈速度要快很多”的人，我建議你們先去學學 JIT 優化程式碼的相關知識。這裡有一個[非常棒的視訊](https://www.youtube.com/watch?v=65-RbBwZQdU)，可能會對你有幫助。

再看一個例子。

```js
// 命令式
var authenticate = function(form) {
  var user = toUser(form);
  return logIn(user);
};

// 宣告式
var authenticate = compose(logIn, toUser);
```

雖然命令式的版本並不一定就是錯的，但還是硬編碼了那種一步接一步的執行方式。而 `compose` 表示式只是簡單地指出了這樣一個事實：使用者驗證是 `toUser` 和 `logIn` 兩個行為的組合。這再次說明，宣告式為潛在的程式碼更新提供了支援，使得我們的應用程式碼成為了一種高階規範（high level specification）。

因為宣告式程式碼不指定執行順序，所以它天然地適合進行並行運算。它與純函式一起解釋了為何函數語言程式設計是未來平行計算的一個不錯選擇——我們真的不需要做什麼就能實現一個並行／併發系統。

## 一個函數式的 flickr

現在我們以一種宣告式的、可組合的方式建立一個示例應用。暫時我們還是會作點小弊，使用副作用；但我們會把副作用的程度降到最低，讓它們與純函式程式碼分離開來。這個示例應用是一個瀏覽器 widget，功能是從 flickr 獲取圖片並在頁面上展示。我們從寫 html 開始：

```html
<!DOCTYPE html>
<html>
  <head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/require.js/2.1.11/require.min.js"></script>
    <script src="flickr.js"></script>
  </head>
  <body></body>
</html>
```

flickr.js 如下：

```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });
    // app goes here
  });
```

這裡我們使用了 [ramda](http://ramdajs.com) ，沒有用 lodash 或者其他類庫。ramda 提供了 `compose`、`curry` 等很多函式。模組載入我們選擇的是 requirejs，我以前用過 requirejs，雖然它有些重，但為了保持一致性，本書將一直使用它。另外，我也把 `trace` 函式寫好了，便於 debug。

有點跑題了。言歸正傳，我們的應用將做 4 件事：

1. 根據特定搜尋關鍵字構造 url
2. 向 flickr 傳送 api 請求
3. 把返回的 json 轉為 html 圖片
4. 把圖片放到螢幕上

注意到沒？上面提到了兩個不純的動作，即從 flickr 的 api 獲取資料和在螢幕上放置圖片這兩件事。我們先來定義這兩個動作，這樣就能隔離它們了。

```js
var Impure = {
  getJSON: _.curry(function(callback, url) {
    $.getJSON(url, callback);
  }),

  setHtml: _.curry(function(sel, html) {
    $(sel).html(html);
  })
};
```

這裡只是簡單地包裝了一下 jQuery 的 `getJSON` 方法，把它變為一個 curry 函式，還有就是把引數位置也調換了下。這些方法都在 `Impure` 名稱空間下，這樣我們就知道它們都是危險函式。在後面的例子中，我們會把這兩個函式變純。

下一步是構造 url 傳給 `Impure.getJSON` 函式。

```js
var url = function (term) {
  return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' + term + '&format=json&jsoncallback=?';
};
```

藉助 monoid 或 combinator （後面會講到這些概念），我們可以使用一些奇技淫巧來讓 `url` 函式變為 pointfree 函式。但是為了可讀性，我們還是選擇以普通的非 pointfree 的方式拼接字串。

讓我們寫一個 `app` 函式傳送請求並把內容放置到螢幕上。

```js
var app = _.compose(Impure.getJSON(trace("response")), url);

app("cats");
```

這會呼叫 `url` 函式，然後把字串傳給 `getJSON` 函式。`getJSON` 已經區域性應用了 `trace`，載入這個應用將會把請求的響應顯示在 console 裡。

<img src="images/console_ss.png"/>

我們想要從這個 json 裡構造圖片，看起來 src 都在 `items` 陣列中的每個 `media` 物件的 `m` 屬性上。

不管怎樣，我們可以使用 ramda 的一個通用 getter 函式 `_.prop()` 來獲取這些巢狀的屬性。不過為了讓你明白這個函式做了什麼事情，我們自己實現一個 prop 看看：

```js
var prop = _.curry(function(property, object){
  return object[property];
});
```

實際上這有點傻，僅僅是用 `[]` 來獲取一個物件的屬性而已。讓我們利用這個函式獲取圖片的 src。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));
```

一旦得到了 `items`，就必須使用 `map` 來分解每一個 url；這樣就得到了一個包含所有 src 的陣列。把它和 `app` 聯結起來，列印結果看看。

```js
var renderImages = _.compose(Impure.setHtml("body"), srcs);
var app = _.compose(Impure.getJSON(renderImages), url);
```

這裡所做的只不過是新建了一個組合，這個組合會呼叫 `srcs` 函式，並把返回結果設定為 body 的 html。我們也把 `trace` 替換為了 `renderImages`，因為已經有了除原始 json 以外的資料。這將會粗暴地把所有的 src 直接顯示在螢幕上。

最後一步是把這些 src 變為真正的圖片。對大型點的應用來說，是應該使用類似 Handlebars 或者 React 這樣的 template/dom 庫來做這件事的。但我們這個應用太小了，只需要一個 img 標籤，所以用 jQuery 就好了。

```js
var img = function (url) {
  return $('<img />', { src: url });
};
```

jQuery 的 `html()` 方法接受標籤陣列為引數，所以我們只須把 src 轉換為 img 標籤然後傳給 `setHtml` 即可。

```js
var images = _.compose(_.map(img), srcs);
var renderImages = _.compose(Impure.setHtml("body"), images);
var app = _.compose(Impure.getJSON(renderImages), url);
```

任務完成！

<img src="images/cats_ss.png" />

下面是完整程式碼：

```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    ////////////////////////////////////////////
    // Utils

    var Impure = {
      getJSON: _.curry(function(callback, url) {
        $.getJSON(url, callback);
      }),

      setHtml: _.curry(function(sel, html) {
        $(sel).html(html);
      })
    };

    var img = function (url) {
      return $('<img />', { src: url });
    };

    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });

    ////////////////////////////////////////////

    var url = function (t) {
      return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' + t + '&format=json&jsoncallback=?';
    };

    var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

    var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

    var images = _.compose(_.map(img), srcs);

    var renderImages = _.compose(Impure.setHtml("body"), images);

    var app = _.compose(Impure.getJSON(renderImages), url);

    app("cats");
  });
```

看看，多麼美妙的宣告式規範啊，只說做什麼，不說怎麼做。現在我們可以把每一行程式碼都視作一個等式，變數名所代表的屬性就是等式的含義。我們可以利用這些屬性去推導分析和重構這個應用。

## 有原則的重構

上面的程式碼是有優化空間的——我們獲取 url map 了一次，把這些 url 變為 img 標籤又 map 了一次。關於 map 和組合是有定律的：

```js
// map 的組合律
var law = compose(map(f), map(g)) == map(compose(f, g));
```

我們可以利用這個定律優化程式碼，進行一次有原則的重構。

```js
// 原有程式碼
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

var images = _.compose(_.map(img), srcs);

```

感謝等式推導（equational reasoning）及純函式的特性，我們可以內聯呼叫 `srcs` 和 `images`，也就是把 map 呼叫排列起來。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(img), _.map(mediaUrl), _.prop('items'));
```

把 `map` 排成一列之後就可以應用組合律了。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(_.compose(img, mediaUrl)), _.prop('items'));
```

現在只需要迴圈一次就可以把每一個物件都轉為 img 標籤了。我們把 map 呼叫的 compose 取出來放到外面，提高一下可讀性。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var mediaToImg = _.compose(img, mediaUrl);

var images = _.compose(_.map(mediaToImg), _.prop('items'));
```

## 總結

我們已經見識到如何在一個小而不失真實的應用中運用新技能了，也已經使用過函數式這個“數學框架”來推導和重構程式碼了。但是異常處理以及程式碼分支呢？如何讓整個應用都是函數式的，而不僅僅是把破壞性的函式放到名稱空間下？如何讓應用更安全更富有表現力？這些都是本書第 2 部分將要解決的問題。

[第 7 章: Hindley-Milner 型別簽名](ch7.md)
