# 第 3 章：純函式的好處

## 再次強調“純”

首先，我們要釐清純函式的概念。

> 純函式是這樣一種函式，即相同的輸入，永遠會得到相同的輸出，而且沒有任何可觀察的副作用。

比如 `slice` 和 `splice`，這兩個函式的作用並無二致——但是注意，它們各自的方式卻大不同，但不管怎麼說作用還是一樣的。我們說 `slice` 符合*純*函式的定義是因為對相同的輸入它保證能返回相同的輸出。而 `splice` 卻會嚼爛呼叫它的那個陣列，然後再吐出來；這就會產生可觀察到的副作用，即這個陣列永久地改變了。

```js
var xs = [1,2,3,4,5];

// 純的
xs.slice(0,3);
//=> [1,2,3]

xs.slice(0,3);
//=> [1,2,3]

xs.slice(0,3);
//=> [1,2,3]


// 不純的
xs.splice(0,3);
//=> [1,2,3]

xs.splice(0,3);
//=> [4,5]

xs.splice(0,3);
//=> []
```

在函數語言程式設計中，我們討厭這種會*改變*資料的笨函式。我們追求的是那種可靠的，每次都能返回同樣結果的函式，而不是像 `splice` 這樣每次呼叫後都把資料弄得一團糟的函式，這不是我們想要的。

來看看另一個例子。

```js
// 不純的
var minimum = 21;

var checkAge = function(age) {
  return age >= minimum;
};


// 純的
var checkAge = function(age) {
  var minimum = 21;
  return age >= minimum;
};
```

在不純的版本中，`checkAge` 的結果將取決於 `minimum` 這個可變變數的值。換句話說，它取決於系統狀態（system state）；這一點令人沮喪，因為它引入了外部的環境，從而增加了認知負荷（cognitive load）。

這個例子可能還不是那麼明顯，但這種依賴狀態是影響系統複雜度的罪魁禍首（http://www.curtclifton.net/storage/papers/MoseleyMarks06a.pdf ）。輸入值之外的因素能夠左右 `checkAge` 的返回值，不僅讓它變得不純，而且導致每次我們思考整個軟體的時候都痛苦不堪。

另一方面，使用純函式的形式，函式就能做到自給自足。我們也可以讓 `minimum` 成為一個不可變（immutable）物件，這樣就能保留純粹性，因為狀態不會有變化。要實現這個效果，必須得建立一個物件，然後呼叫 `Object.freeze` 方法：

```js
var immutableState = Object.freeze({
  minimum: 21
});
```

## 副作用可能包括...

讓我們來仔細研究一下“副作用”以便加深理解。那麼，我們在*純函式*定義中提到的萬分邪惡的*副作用*到底是什麼？“作用”我們可以理解為一切除結果計算之外發生的事情。

“作用”本身並沒什麼壞處，而且在本書後面的章節你隨處可見它的身影。“副作用”的關鍵部分在於“副”。就像一潭死水中的“水”本身並不是幼蟲的培養器，“死”才是生成蟲群的原因。同理，副作用中的“副”是滋生 bug 的溫床。

> *副作用*是在計算結果的過程中，系統狀態的一種變化，或者與外部世界進行的*可觀察的互動*。

副作用可能包含，但不限於：

  * 更改檔案系統
  * 往資料庫插入記錄
  * 傳送一個 http 請求
  * 可變資料
  * 列印/log
  * 獲取使用者輸入
  * DOM 查詢
  * 訪問系統狀態

這個列表還可以繼續寫下去。概括來講，只要是跟函式外部環境發生的互動就都是副作用——這一點可能會讓你懷疑無副作用程式設計的可行性。函數語言程式設計的哲學就是假定副作用是造成不正當行為的主要原因。

這並不是說，要禁止使用一切副作用，而是說，要讓它們在可控的範圍內發生。後面講到 functor 和 monad 的時候我們會學習如何控制它們，目前還是儘量遠離這些陰險的函式為好。

副作用讓一個函式變得不*純*是有道理的：從定義上來說，純函式必須要能夠根據相同的輸入返回相同的輸出；如果函式需要跟外部事物打交道，那麼就無法保證這一點了。

我們來仔細瞭解下為何要堅持這種「相同輸入得到相同輸出」原則。注意，我們要複習一些八年級數學知識了。

## 八年級數學

根據 mathisfun.com：

> 函式是不同數值之間的特殊關係：每一個輸入值返回且只返回一個輸出值。

換句話說，函式只是兩種數值之間的關係：輸入和輸出。儘管每個輸入都只會有一個輸出，但不同的輸入卻可以有相同的輸出。下圖展示了一個合法的從 `x` 到 `y` 的函式關係；

<img src="images/function-sets.gif" />（http://www.mathsisfun.com/sets/function.html）

相反，下面這張圖表展示的就*不是*一種函式關係，因為輸入值 `5` 指向了多個輸出：

<img src="images/relation-not-function.gif" />（http://www.mathsisfun.com/sets/function.html）

函式可以描述為一個集合，這個集合裡的內容是 (輸入, 輸出) 對：`[(1,2), (3,6), (5,10)]`（看起來這個函式是把輸入值加倍）。

或者一張表：

<table> <tr> <th>輸入</th> <th>輸出</th> </tr> <tr> <td>1</td> <td>2</td> </tr> <tr> <td>2</td> <td>4</td> </tr> <tr> <td>3</td> <td>6</td> </tr> </table>

甚至一個以 `x` 為輸入 `y` 為輸出的函式曲線圖：

<img src="images/fn_graph.png" width="300" height="300" />

如果輸入直接指明瞭輸出，那麼就沒有必要再實現具體的細節了。因為函式僅僅只是輸入到輸出的對映而已，所以簡單地寫一個物件就能“執行”它，使用 `[]` 代替 `()` 即可。

```js
var toLowerCase = {"A":"a", "B": "b", "C": "c", "D": "d", "E": "e", "D": "d"};

toLowerCase["C"];
//=> "c"

var isPrime = {1:false, 2: true, 3: true, 4: false, 5: true, 6:false};

isPrime[3];
//=> true
```

當然了，實際情況中你可能需要進行一些計算而不是手動指定各項值；不過上例倒是表明了另外一種思考函式的方式。（你可能會想“要是函式有多個引數呢？”。的確，這種情況表明了以數學方式思考問題的一點點不便。暫時我們可以把它們打包放到數組裡，或者把 `arguments` 物件看成是輸入。等學習 `curry` 的概念之後，你就知道如何直接為函式在數學上的定義建模了。）

戲劇性的是：純函式*就是*數學上的函式，而且是函數語言程式設計的全部。使用這些純函式程式設計能夠帶來大量的好處，讓我們來看一下為何要不遺餘力地保留函式的純粹性的原因。

## 追求“純”的理由

### 可快取性（Cacheable）

首先，純函式總能夠根據輸入來做快取。實現快取的一種典型方式是 memoize 技術：

```js
var squareNumber  = memoize(function(x){ return x*x; });

squareNumber(4);
//=> 16

squareNumber(4); // 從快取中讀取輸入值為 4 的結果
//=> 16

squareNumber(5);
//=> 25

squareNumber(5); // 從快取中讀取輸入值為 5 的結果
//=> 25
```

下面的程式碼是一個簡單的實現，儘管它不太健壯。

```js
var memoize = function(f) {
  var cache = {};

  return function() {
    var arg_str = JSON.stringify(arguments);
    cache[arg_str] = cache[arg_str] || f.apply(f, arguments);
    return cache[arg_str];
  };
};
```

值得注意的一點是，可以通過延遲執行的方式把不純的函式轉換為純函式：

```js
var pureHttpCall = memoize(function(url, params){
  return function() { return $.getJSON(url, params); }
});
```

這裡有趣的地方在於我們並沒有真正傳送 http 請求——只是返回了一個函式，當呼叫它的時候才會發請求。這個函式之所以有資格成為純函式，是因為它總是會根據相同的輸入返回相同的輸出：給定了 `url` 和 `params` 之後，它就只會返回同一個傳送 http 請求的函式。

我們的 `memoize` 函式工作起來沒有任何問題，雖然它快取的並不是 http 請求所返回的結果，而是生成的函式。

現在來看這種方式意義不大，不過很快我們就會學習一些技巧來發掘它的用處。重點是我們可以快取任意一個函式，不管它們看起來多麼具有破壞性。

### 可移植性／自文件化（Portable / Self-Documenting）

純函式是完全自給自足的，它需要的所有東西都能輕易獲得。仔細思考思考這一點...這種自給自足的好處是什麼呢？首先，純函式的依賴很明確，因此更易於觀察和理解——沒有偷偷摸摸的小動作。

```js
// 不純的
var signUp = function(attrs) {
  var user = saveUser(attrs);
  welcomeUser(user);
};

var saveUser = function(attrs) {
    var user = Db.save(attrs);
    ...
};

var welcomeUser = function(user) {
    Email(user, ...);
    ...
};

// 純的
var signUp = function(Db, Email, attrs) {
  return function() {
    var user = saveUser(Db, attrs);
    welcomeUser(Email, user);
  };
};

var saveUser = function(Db, attrs) {
    ...
};

var welcomeUser = function(Email, user) {
    ...
};
```

這個例子表明，純函式對於其依賴必須要誠實，這樣我們就能知道它的目的。僅從純函式版本的 `signUp` 的簽名就可以看出，它將要用到 `Db`、`Email` 和 `attrs`，這在最小程度上給了我們足夠多的資訊。

後面我們會學習如何不通過這種僅僅是延遲執行的方式來讓一個函式變純，不過這裡的重點應該很清楚，那就是相比不純的函式，純函式能夠提供多得多的資訊；前者天知道它們暗地裡都幹了些什麼。

其次，通過強迫“注入”依賴，或者把它們當作引數傳遞，我們的應用也更加靈活；因為資料庫或者郵件客戶端等等都引數化了（別擔心，我們有辦法讓這種方式不那麼單調乏味）。如果要使用另一個 `Db`，只需把它傳給函式就行了。如果想在一個新應用中使用這個可靠的函式，儘管把新的 `Db` 和 `Email` 傳遞過去就好了，非常簡單。

在 JavaScript 的設定中，可移植性可以意味著把函式序列化（serializing）並通過 socket 傳送。也可以意味著程式碼能夠在 web workers 中執行。總之，可移植性是一個非常強大的特性。

指令式程式設計中“典型”的方法和過程都深深地根植於它們所在的環境中，通過狀態、依賴和有效作用（available effects）達成；純函式與此相反，它與環境無關，只要我們願意，可以在任何地方執行它。

你上一次把某個類方法拷貝到新的應用中是什麼時候？我最喜歡的名言之一是 Erlang 語言的作者 Joe Armstrong 說的這句話：“面嚮物件語言的問題是，它們永遠都要隨身攜帶那些隱式的環境。你只需要一個香蕉，但卻得到一個拿著香蕉的大猩猩...以及整個叢林”。

### 可測試性（Testable）

第三點，純函式讓測試更加容易。我們不需要偽造一個“真實的”支付閘道器，或者每一次測試之前都要配置、之後都要斷言狀態（assert the state）。只需簡單地給函式一個輸入，然後斷言輸出就好了。

事實上，我們發現函數語言程式設計的社群正在開創一些新的測試工具，能夠幫助我們自動生成輸入並斷言輸出。這超出了本書範圍，但是我強烈推薦你去試試 *Quickcheck*——一個為函式式環境量身定製的測試工具。

### 合理性（Reasonable）

很多人相信使用純函式最大的好處是*引用透明性*（referential transparency）。如果一段程式碼可以替換成它執行所得的結果，而且是在不改變整個程式行為的前提下替換的，那麼我們就說這段程式碼是引用透明的。

由於純函式總是能夠根據相同的輸入返回相同的輸出，所以它們就能夠保證總是返回同一個結果，這也就保證了引用透明性。我們來看一個例子。

```js
var Immutable = require('immutable');

var decrementHP = function(player) {
  return player.set("hp", player.hp-1);
};

var isSameTeam = function(player1, player2) {
  return player1.team === player2.team;
};

var punch = function(player, target) {
  if(isSameTeam(player, target)) {
    return target;
  } else {
    return decrementHP(target);
  }
};

var jobe = Immutable.Map({name:"Jobe", hp:20, team: "red"});
var michael = Immutable.Map({name:"Michael", hp:20, team: "green"});

punch(jobe, michael);
//=> Immutable.Map({name:"Michael", hp:19, team: "green"})
```

`decrementHP`、`isSameTeam` 和 `punch` 都是純函式，所以是引用透明的。我們可以使用一種叫做“等式推導”（equational reasoning）的技術來分析程式碼。所謂“等式推導”就是“一對一”替換，有點像在不考慮程式性執行的怪異行為（quirks of programmatic evaluation）的情況下，手動執行相關程式碼。我們藉助引用透明性來剖析一下這段程式碼。

首先內聯 `isSameTeam` 函式：

```js
var punch = function(player, target) {
  if(player.team === target.team) {
    return target;
  } else {
    return decrementHP(target);
  }
};
```

因為是不可變資料，我們可以直接把 `team` 替換為實際值：

```js
var punch = function(player, target) {
  if("red" === "green") {
    return target;
  } else {
    return decrementHP(target);
  }
};
```

`if` 語句執行結果為 `false`，所以可以把整個 `if` 語句都刪掉：

```js
var punch = function(player, target) {
  return decrementHP(target);
};
```

如果再內聯 `decrementHP`，我們會發現這種情況下，`punch` 變成了一個讓 `hp` 的值減 1 的呼叫：

```js
var punch = function(player, target) {
  return target.set("hp", target.hp-1);
};
```

總之，等式推導帶來的分析程式碼的能力對重構和理解程式碼非常重要。事實上，我們重構海鷗程式使用的正是這項技術：利用加和乘的特性。對這些技術的使用將會貫穿本書，真的。

### 並行程式碼

最後一點，也是決定性的一點：我們可以並行執行任意純函式。因為純函式根本不需要訪問共享的記憶體，而且根據其定義，純函式也不會因副作用而進入競爭態（race condition）。

並行程式碼在服務端 js 環境以及使用了 web worker 的瀏覽器那裡是非常容易實現的，因為它們使用了執行緒（thread）。不過出於對非純函式複雜度的考慮，當前主流觀點還是避免使用這種並行。

## 總結

我們已經瞭解什麼是純函數了，也看到作為函式式程式設計師的我們，為何深信純函式是不同凡響的。從這開始，我們將盡力以純函式式的方式書寫所有的函式。為此我們將需要一些額外的工具來達成目標，同時也儘量把非純函式從純函式程式碼中分離。

如果手頭沒有一些工具，那麼純函式程式寫起來就有點費力。我們不得不玩雜耍似的通過到處傳遞引數來操作資料，而且還被禁止使用狀態，更別說“作用”了。沒有人願意這樣自虐。所以讓我們來學習一個叫 curry 的新工具。

[第 4 章: 柯里化（curry）](ch4.md)
