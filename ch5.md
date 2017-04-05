# 第 5 章: 程式碼組合（compose）

## 函式飼養

這就是 `組合`（compose，以下將稱之為組合）：

```js
var compose = function(f,g) {
  return function(x) {
    return f(g(x));
  };
};
```

`f` 和 `g` 都是函式，`x` 是在它們之間通過“管道”傳輸的值。

`組合`看起來像是在飼養函式。你就是飼養員，選擇兩個有特點又遭你喜歡的函式，讓它們結合，產下一個嶄新的函式。組合的用法如下：

```js
var toUpperCase = function(x) { return x.toUpperCase(); };
var exclaim = function(x) { return x + '!'; };
var shout = compose(exclaim, toUpperCase);

shout("send in the clowns");
//=> "SEND IN THE CLOWNS!"
```

兩個函式組合之後返回了一個新函式是完全講得通的：組合某種型別（本例中是函式）的兩個元素本就該生成一個該型別的新元素。把兩個樂高積木組合起來絕不可能得到一個林肯積木。所以這是有道理的，我們將在適當的時候探討這方面的一些底層理論。

在 `compose` 的定義中，`g` 將先於 `f` 執行，因此就建立了一個從右到左的資料流。這樣做的可讀性遠遠高於巢狀一大堆的函式呼叫，如果不用組合，`shout` 函式將會是這樣的：

```js
var shout = function(x){
  return exclaim(toUpperCase(x));
};
```

讓程式碼從右向左執行，而不是由內而外執行，我覺得可以稱之為“左傾”（籲——）。我們來看一個順序很重要的例子：

```js
var head = function(x) { return x[0]; };
var reverse = reduce(function(acc, x){ return [x].concat(acc); }, []);
var last = compose(head, reverse);

last(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'uppercut'
```

`reverse` 反轉列表，`head` 取列表中的第一個元素；所以結果就是得到了一個 `last` 函式（譯者注：即取列表的最後一個元素），雖然它效能不高。這個組合中函式的執行順序應該是顯而易見的。儘管我們可以定義一個從左向右的版本，但是從右向左執行更加能夠反映數學上的含義——是的，組合的概念直接來自於數學課本。實際上，現在是時候去看看所有的組合都有的一個特性了。

```js
// 結合律（associativity）
var associative = compose(f, compose(g, h)) == compose(compose(f, g), h);
// true
```

這個特性就是結合律，符合結合律意味著不管你是把 `g` 和 `h` 分到一組，還是把 `f` 和 `g` 分到一組都不重要。所以，如果我們想把字串變為大寫，可以這麼寫：

```js
compose(toUpperCase, compose(head, reverse));

// 或者
compose(compose(toUpperCase, head), reverse);
```

因為如何為 `compose` 的呼叫分組不重要，所以結果都是一樣的。這也讓我們有能力寫一個可變的組合（variadic compose），用法如下：

```js
// 前面的例子中我們必須要寫兩個組合才行，但既然組合是符合結合律的，我們就可以只寫一個，
// 而且想傳給它多少個函式就傳給它多少個，然後讓它自己決定如何分組。

var lastUpper = compose(toUpperCase, head, reverse);

lastUpper(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'UPPERCUT'


var loudLastUpper = compose(exclaim, toUpperCase, head, reverse)

loudLastUpper(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'UPPERCUT!'
```

運用結合律能為我們帶來強大的靈活性，還有對執行結果不會出現意外的那種平和心態。至於稍微複雜些的可變組合，也都包含在本書的 `support` 庫裡了，而且你也可以在類似 [lodash][lodash-website]、[underscore][underscore-website] 以及 [ramda][ramda-website] 這樣的類庫中找到它們的常規定義。

結合律的一大好處是任何一個函式分組都可以被拆開來，然後再以它們自己的組合方式打包在一起。讓我們來重構重構前面的例子：

```js
var loudLastUpper = compose(exclaim, toUpperCase, head, reverse);

// 或
var last = compose(head, reverse);
var loudLastUpper = compose(exclaim, toUpperCase, last);

// 或
var last = compose(head, reverse);
var angry = compose(exclaim, toUpperCase);
var loudLastUpper = compose(angry, last);

// 更多變種...
```

關於如何組合，並沒有標準的答案——我們只是以自己喜歡的方式搭樂高積木罷了。通常來說，最佳實踐是讓組合可重用，就像 `last` 和 `angry` 那樣。如果熟悉 Fowler 的《[重構][refactoring-book]》一書的話，你可能會認識到這個過程叫做 “[extract method][extract-method-refactor]”——只不過不需要關心物件的狀態。

## pointfree

pointfree 模式指的是，永遠不必說出你的資料。咳咳對不起（譯者注：此處原文是“Pointfree style means never having to say your data”，源自 1970 年的電影 *Love Story* 裡的一句著名臺詞“Love means never having to say you're sorry”。緊接著作者又說了一句“Excuse me”，大概是一種幽默）。它的意思是說，函式無須提及將要操作的資料是什麼樣的。一等公民的函式、柯里化（curry）以及組合協作起來非常有助於實現這種模式。

```js
// 非 pointfree，因為提到了資料：word
var snakeCase = function (word) {
  return word.toLowerCase().replace(/\s+/ig, '_');
};

// pointfree
var snakeCase = compose(replace(/\s+/ig, '_'), toLowerCase);
```

看到 `replace` 是如何被區域性呼叫的了麼？這裡所做的事情就是通過管道把資料在接受單個引數的函式間傳遞。利用 curry，我們能夠做到讓每個函式都先接收資料，然後操作資料，最後再把資料傳遞到下一個函式那裡去。另外注意在 pointfree 版本中，不需要 `word` 引數就能建構函式；而在非 pointfree 的版本中，必須要有 `word` 才能進行進行一切操作。

我們再來看一個例子。

```js
// 非 pointfree，因為提到了資料：name
var initials = function (name) {
  return name.split(' ').map(compose(toUpperCase, head)).join('. ');
};

// pointfree
var initials = compose(join('. '), map(compose(toUpperCase, head)), split(' '));

initials("hunter stockton thompson");
// 'H. S. T'
```

另外，pointfree 模式能夠幫助我們減少不必要的命名，讓程式碼保持簡潔和通用。對函式式程式碼來說，pointfree 是非常好的石蕊試驗，因為它能告訴我們一個函式是否是接受輸入返回輸出的小函式。比如，while 迴圈是不能組合的。不過你也要警惕，pointfree 就像是一把雙刃劍，有時候也能混淆視聽。並非所有的函式式程式碼都是 pointfree 的，不過這沒關係。可以使用它的時候就使用，不能使用的時候就用普通函式。

## debug

組合的一個常見錯誤是，在沒有區域性呼叫之前，就組合類似 `map` 這樣接受兩個引數的函式。

```js
// 錯誤做法：我們傳給了 `angry` 一個數組，根本不知道最後傳給 `map` 的是什麼東西。
var latin = compose(map, angry, reverse);

latin(["frog", "eyes"]);
// error


// 正確做法：每個函式都接受一個實際引數。
var latin = compose(map(angry), reverse);

latin(["frog", "eyes"]);
// ["EYES!", "FROG!"])
```

如果在 debug 組合的時候遇到了困難，那麼可以使用下面這個實用的，但是不純的 `trace` 函式來追蹤程式碼的執行情況。

```js
var trace = curry(function(tag, x){
  console.log(tag, x);
  return x;
});

var dasherize = compose(join('-'), toLower, split(' '), replace(/\s{2,}/ig, ' '));

dasherize('The world is a vampire');
// TypeError: Cannot read property 'apply' of undefined
```

這裡報錯了，來 `trace` 下：

```js
var dasherize = compose(join('-'), toLower, trace("after split"), split(' '), replace(/\s{2,}/ig, ' '));
// after split [ 'The', 'world', 'is', 'a', 'vampire' ]
```

啊！`toLower` 的引數是一個數組，所以需要先用 `map` 呼叫一下它。

```js
var dasherize = compose(join('-'), map(toLower), split(' '), replace(/\s{2,}/ig, ' '));

dasherize('The world is a vampire');

// 'the-world-is-a-vampire'
```

`trace` 函式允許我們在某個特定的點觀察資料以便 debug。像 haskell 和 purescript 之類的語言出於開發的方便，也都提供了類似的函式。

組合將成為我們構造程式的工具，而且幸運的是，它背後是有一個強大的理論做支撐的。讓我們來研究研究這個理論。

## 範疇學

範疇學（category theory）是數學中的一個抽象分支，能夠形式化諸如集合論（set theory）、型別論（type theory）、群論（group theory）以及邏輯學（logic）等數學分支中的一些概念。範疇學主要處理物件（object）、態射（morphism）和變化式（transformation），而這些概念跟程式設計的聯絡非常緊密。下圖是一些相同的概念分別在不同理論下的形式：

<img src="images/cat_theory.png" />

抱歉，我沒有任何要嚇唬你的意思。我並不假設你對這些概念都瞭如指掌，我只是想讓你明白這裡面有多少重複的內容，讓你知道為何範疇學要統一這些概念。

在範疇學中，有一個概念叫做...範疇。有著以下這些元件（component）的蒐集（collection）就構成了一個範疇：

  * 物件的蒐集
  * 態射的蒐集
  * 態射的組合
  * identity 這個獨特的態射

範疇學抽象到足以模擬任何事物，不過目前我們最關心的還是型別和函式，所以讓我們把範疇學運用到它們身上看看。

**物件的蒐集**

物件就是資料型別，例如 `String`、`Boolean`、`Number` 和 `Object` 等等。通常我們把資料型別視作所有可能的值的一個集合（set）。像 `Boolean` 就可以看作是 `[true, false]` 的集合，`Number` 可以是所有實數的一個集合。把型別當作集合對待是有好處的，因為我們可以利用集合論（set theory）處理型別。

**態射的蒐集**

態射是標準的、普通的純函式。

**態射的組合**

你可能猜到了，這就是本章介紹的新玩意兒——`組合`。我們已經討論過 `compose` 函式是符合結合律的，這並非巧合，結合律是在範疇學中對任何組合都適用的一個特性。

這張圖展示了什麼是組合：

<img src="images/cat_comp1.png" />
<img src="images/cat_comp2.png" />

這裡有一個具體的例子：

```js
var g = function(x){ return x.length; };
var f = function(x){ return x === 4; };
var isFourLetterWord = compose(f, g);
```

**identity 這個獨特的態射**

讓我們介紹一個名為 `id` 的實用函式。這個函式接受隨便什麼輸入然後原封不動地返回它：

```js
var id = function(x){ return x; };
```

你可能會問“這到底哪裡有用了？”。別急，我們會在隨後的章節中拓展這個函式的，暫時先把它當作一個可以替代給定值的函式——一個假裝自己是普通資料的函式。

`id` 函式跟組合一起使用簡直完美。下面這個特性對所有的一元函式（unary function）（一元函式：只接受一個引數的函式） `f` 都成立：

```js
// identity
compose(id, f) == compose(f, id) == f;
// true
```

嘿，這就是實數的單位元（identity property）嘛！如果這還不夠清楚直白，彆著急，慢慢理解它的無用性。很快我們就會到處使用 `id` 了，不過暫時我們還是把當作一個替代給定值的函式。這對寫 pointfree 的程式碼非常有用。

好了，以上就是型別和函式的範疇。不過如果你是第一次聽說這些概念，我估計你還是有些迷糊，不知道範疇到底是什麼，為什麼有用。沒關係，本書全書都在藉助這些知識編寫示例程式碼。至於現在，就在本章，本行文字中，你至少可以認為它向我們提供了有關組合的知識——比如結合律和單位律。

除了型別和函式，還有什麼範疇呢？還有很多，比如我們可以定義一個有向圖（directed graph），以節點為物件，以邊為態射，以路徑連線為組合。還可以定義一個實數型別（Number），以所有的實數物件，以 `>=` 為態射（實際上任何偏序（partial order）或全序（total order）都可以成為一個範疇）。範疇的總數是無限的，但是要達到本書的目的，我們只需要關心上面定義的範疇就好了。至此我們已經大致瀏覽了一些表面的東西，必須要繼續後面的內容了。

## 總結

組合像一系列管道那樣把不同的函式聯絡在一起，資料就可以也必須在其中流動——畢竟純函式就是輸入對輸出，所以打破這個鏈條就是不尊重輸出，就會讓我們的應用一無是處。

我們認為組合是高於其他所有原則的設計原則，這是因為組合讓我們的程式碼簡單而富有可讀性。另外範疇學將在應用架構、模擬副作用和保證正確性方面扮演重要角色。

現在我們已經有足夠的知識去進行一些實際的練習了，讓我們來編寫一個示例應用。

[第 6 章: 示例應用](ch6.md)

## 練習

```js
require('../../support');
var _ = require('ramda');
var accounting = require('accounting');

// 示例資料
var CARS = [
    {name: "Ferrari FF", horsepower: 660, dollar_value: 700000, in_stock: true},
    {name: "Spyker C12 Zagato", horsepower: 650, dollar_value: 648000, in_stock: false},
    {name: "Jaguar XKR-S", horsepower: 550, dollar_value: 132000, in_stock: false},
    {name: "Audi R8", horsepower: 525, dollar_value: 114200, in_stock: false},
    {name: "Aston Martin One-77", horsepower: 750, dollar_value: 1850000, in_stock: true},
    {name: "Pagani Huayra", horsepower: 700, dollar_value: 1300000, in_stock: false}
  ];

// 練習 1:
// ============
// 使用 _.compose() 重寫下面這個函式。提示：_.prop() 是 curry 函式
var isLastInStock = function(cars) {
  var last_car = _.last(cars);
  return _.prop('in_stock', last_car);
};

// 練習 2:
// ============
// 使用 _.compose()、_.prop() 和 _.head() 獲取第一個 car 的 name
var nameOfFirstCar = undefined;


// 練習 3:
// ============
// 使用幫助函式 _average 重構 averageDollarValue 使之成為一個組合
var _average = function(xs) { return reduce(add, 0, xs) / xs.length; }; // <- 無須改動

var averageDollarValue = function(cars) {
  var dollar_values = map(function(c) { return c.dollar_value; }, cars);
  return _average(dollar_values);
};


// 練習 4:
// ============
// 使用 compose 寫一個 sanitizeNames() 函式，返回一個下劃線連線的小寫字串：例如：sanitizeNames(["Hello World"]) //=> ["hello_world"]。

var _underscore = replace(/\W+/g, '_'); //<-- 無須改動，並在 sanitizeNames 中使用它

var sanitizeNames = undefined;


// 彩蛋 1:
// ============
// 使用 compose 重構 availablePrices

var availablePrices = function(cars) {
  var available_cars = _.filter(_.prop('in_stock'), cars);
  return available_cars.map(function(x){
    return accounting.formatMoney(x.dollar_value);
  }).join(', ');
};


// 彩蛋 2:
// ============
// 重構使之成為 pointfree 函式。提示：可以使用 _.flip()

var fastestCar = function(cars) {
  var sorted = _.sortBy(function(car){ return car.horsepower }, cars);
  var fastest = _.last(sorted);
  return fastest.name + ' is the fastest';
};
```

[lodash-website]: https://lodash.com/
[underscore-website]: http://underscorejs.org/
[ramda-website]: http://ramdajs.com/
[refactoring-book]: http://martinfowler.com/books/refactoring.html
[extract-method-refactor]: http://refactoring.com/catalog/extractMethod.html
