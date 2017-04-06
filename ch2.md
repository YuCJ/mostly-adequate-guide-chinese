# 第 2 章: 一等公民的函式

## 快速概覽

當我們說函式是“一等公民”的時候，我們實際上說的是它們和其他物件都一樣...所以就是普通公民（坐經濟艙的人？）。函式真沒什麼特殊的，你可以像對待任何其他資料型別一樣對待它們——把它們存在數組裡，當作引數傳遞，賦值給變數...等等。

這是 JavaScript 語言的基礎概念，不過還是值得提一提的，因為在 Github 上隨便一搜就能看到對這個概念的集體無視，或者也可能是無知。我們來看一個杜撰的例子：

```js
var hi = function(name){
  return "Hi " + name;
};

var greeting = function(name) {
  return hi(name);
};
```

這裡 `greeting` 指向的那個把 `hi` 包了一層的包裹函式完全是多餘的。為什麼？因為 JavaScript 的函式是*可呼叫*的，當 `hi` 後面緊跟 `()` 的時候就會執行並返回一個值；如果沒有 `()`，`hi` 就簡單地返回存到這個變數裡的函式。我們來確認一下：

```js
hi;
// function(name){
//  return "Hi " + name
// }

hi("jonas");
// "Hi jonas"
```

`greeting` 只不過是轉了個身然後以相同的引數呼叫了 `hi` 函式而已，因此我們可以這麼寫：

```js
var greeting = hi;


greeting("times");
// "Hi times"
```

換句話說，`hi` 已經是個接受一個引數的函數了，為何要再定義一個額外的包裹函式，而它僅僅是用這個相同的引數呼叫 `hi`？完全沒有道理。這就像在大夏天裡穿上你最厚的大衣，只是為了跟熱空氣過不去，然後吃上個冰棍。真是脫褲子放屁多此一舉。

用一個函式把另一個函式包起來，目的僅僅是延遲執行，真的是非常糟糕的程式設計習慣。（稍後我將告訴你原因，跟可維護性密切相關。）

充分理解這個問題對讀懂本書後面的內容至關重要，所以我們再來看幾個例子。以下程式碼都來自 npm 上的模組包：

```js
// 太傻了
var getServerStuff = function(callback){
  return ajaxCall(function(json){
    return callback(json);
  });
};

// 這才像樣
var getServerStuff = ajaxCall;
```

世界上到處都充斥著這樣的垃圾 ajax 程式碼。以下是上述兩種寫法等價的原因：

```js
// 這行
return ajaxCall(function(json){
  return callback(json);
});

// 等價於這行
return ajaxCall(callback);

// 那麼，重構下 getServerStuff
var getServerStuff = function(callback){
  return ajaxCall(callback);
};

// ...就等於
var getServerStuff = ajaxCall; // <-- 看，沒有括號哦
```

各位，以上才是寫函式的正確方式。一會兒再告訴你為何我對此如此執著。

```js
var BlogController = (function() {
  var index = function(posts) {
    return Views.index(posts);
  };

  var show = function(post) {
    return Views.show(post);
  };

  var create = function(attrs) {
    return Db.create(attrs);
  };

  var update = function(post, attrs) {
    return Db.update(post, attrs);
  };

  var destroy = function(post) {
    return Db.destroy(post);
  };

  return {index: index, show: show, create: create, update: update, destroy: destroy};
})();
```

這個可笑的控制器（controller）99% 的程式碼都是垃圾。我們可以把它重寫成這樣：

```js
var BlogController = {index: Views.index, show: Views.show, create: Db.create, update: Db.update, destroy: Db.destroy};
```

...或者直接全部刪掉，因為它的作用僅僅就是把檢視（Views）和資料庫（Db）打包在一起而已。

## 為何鍾愛一等公民？

好了，現在我們來看看鐘愛一等公民的原因是什麼。前面 `getServerStuff` 和 `BlogController` 兩個例子你也都看到了，雖說新增一些沒有實際用處的間接層實現起來很容易，但這樣做除了徒增程式碼量，提高維護和檢索程式碼的成本外，沒有任何用處。

另外，如果一個函式被不必要地包裹起來了，而且發生了改動，那麼包裹它的那個函式也要做相應的變更。

```js
httpGet('/post/2', function(json){
  return renderPost(json);
});
```

如果 `httpGet` 要改成可以丟擲一個可能出現的 `err` 異常，那我們還要回過頭去把“膠水”函式也改了。

```js
// 把整個應用裡的所有 httpGet 呼叫都改成這樣，可以傳遞 err 引數。
httpGet('/post/2', function(json, err){
  return renderPost(json, err);
});
```

寫成一等公民函式的形式，要做的改動將會少得多：

```js
httpGet('/post/2', renderPost);  // renderPost 將會在 httpGet 中呼叫，想要多少引數都行
```

除了刪除不必要的函式，正確地為引數命名也必不可少。當然命名不是什麼大問題，但還是有可能存在一些不當的命名，尤其隨著程式碼量的增長以及需求的變更，這種可能性也會增加。

專案中常見的一種造成混淆的原因是，針對同一個概念使用不同的命名。還有通用程式碼的問題。比如，下面這兩個函式做的事情一模一樣，但後一個就顯得更加通用，可重用性也更高：

```js
// 只針對當前的部落格
var validArticles = function(articles) {
  return articles.filter(function(article){
    return article !== null && article !== undefined;
  });
};

// 對未來的專案友好太多
var compact = function(xs) {
  return xs.filter(function(x) {
    return x !== null && x !== undefined;
  });
};
```

在命名的時候，我們特別容易把自己限定在特定的資料上（本例中是 `articles`）。這種現象很常見，也是重複造輪子的一大原因。

有一點我必須得指出，你一定要非常小心 `this` 值，別讓它反咬你一口，這一點與物件導向程式碼類似。如果一個底層函式使用了 `this`，而且是以一等公民的方式被呼叫的，那你就等著 JS 這個蹩腳的抽象概念發怒吧。

```js
var fs = require('fs');

// 太可怕了
fs.readFile('freaky_friday.txt', Db.save);

// 好一點點
fs.readFile('freaky_friday.txt', Db.save.bind(Db));

```

把 Db 繫結（bind）到它自己身上以後，你就可以隨心所欲地呼叫它的原型鏈式垃圾程式碼了。`this` 就像一塊髒尿布，我儘可能地避免使用它，因為在函數語言程式設計中根本用不到它。然而，在使用其他的類庫時，你卻不得不向這個瘋狂的世界低頭。

也有人反駁說 `this` 能提高執行速度。如果你是這種對速度吹毛求疵的人，那你還是合上這本書吧。要是沒法退貨退款，也許你可以去換一本更入門的書來讀。

至此，我們才準備好繼續後面的章節。

[第 3 章: 純函式的好處](ch3.md)
