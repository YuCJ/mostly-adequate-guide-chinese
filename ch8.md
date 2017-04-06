# 特百惠

（譯者注：特百惠是美國家居用品品牌，代表產品是塑料容器。）

## 強大的容器

<img src="images/jar.jpg" alt="http://blog.dwinegar.com/2011/06/another-jar.html" />

我們已經知道如何書寫函數式的程式了，即通過管道把資料在一系列純函式間傳遞的程式。我們也知道了，這些程式就是宣告式的行為規範。但是，控制流（control flow）、異常處理（error handling）、非同步操作（asynchronous actions）和狀態（state）呢？還有更棘手的作用（effects）呢？本章將對上述這些抽象概念賴以建立的基礎作一番探究。

首先我們將建立一個容器（container）。這個容器必須能夠裝載任意型別的值；否則的話，像只能裝木薯布丁的密封塑料袋是沒什麼用的。這個容器將會是一個物件，但我們不會為它新增物件導向觀念下的屬性和方法。是的，我們將把它當作一個百寶箱——一個存放寶貴的資料的特殊盒子。

```js
var Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };
```

這是本書的第一個容器，我們貼心地把它命名為 `Container`。我們將使用 `Container.of` 作為構造器（constructor），這樣就不用到處去寫糟糕的 `new` 關鍵字了，非常省心。實際上不能這麼簡單地看待 `of` 函式，但暫時先認為它是把值放到容器裡的一種方式。

我們來檢驗下這個嶄新的盒子：

```js
Container.of(3)
//=> Container(3)


Container.of("hotdogs")
//=> Container("hotdogs")


Container.of(Container.of({name: "yoda"}))
//=> Container(Container({name: "yoda" }))
```

如果用的是 node，那麼你會看到打印出來的是 `{__value: x}`，而不是實際值 `Container(x)`；Chrome 打印出來的是正確的。不過這並不重要，只要你理解 `Container` 是什麼樣的就行了。有些環境下，你也可以重寫 `inspect` 方法，但我們不打算涉及這方面的知識。在本書中，出於教學和美學上的考慮，我們將把概念性的輸出都寫成好像 `inspect` 被重寫了的樣子，因為這樣寫的教育意義將遠遠大於 `{__value: x}`。

在繼續後面的內容之前，先澄清幾點：

* `Container` 是個只有一個屬性的物件。儘管容器可以有不止一個的屬性，但大多數容器還是隻有一個。我們很隨意地把 `Container` 的這個屬性命名為 `__value`。
* `__value` 不能是某個特定的型別，不然 `Container` 就對不起它這個名字了。
* 資料一旦存放到 `Container`，就會一直待在那兒。我們*可以*用 `.__value` 獲取到資料，但這樣做有悖初衷。

如果把容器想象成玻璃罐的話，上面這三條陳述的理由就會比較清晰了。但是暫時，請先保持耐心。

## 第一個 functor

一旦容器裡有了值，不管這個值是什麼，我們就需要一種方法來讓別的函式能夠操作它。

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function(f){
  return Container.of(f(this.__value))
}
```

這個 `map` 跟陣列那個著名的 `map` 一樣，除了前者的引數是 `Container a` 而後者是 `[a]`。它們的使用方式也幾乎一致：

```js
Container.of(2).map(function(two){ return two + 2 })
//=> Container(4)


Container.of("flamethrowers").map(function(s){ return s.toUpperCase() })
//=> Container("FLAMETHROWERS")


Container.of("bombs").map(concat(' away')).map(_.prop('length'))
//=> Container(10)
```

為什麼要使用這樣一種方法？因為我們能夠在不離開 `Container` 的情況下操作容器裡面的值。這是非常了不起的一件事情。`Container` 裡的值傳遞給 `map` 函式之後，就可以任我們操作；操作結束後，為了防止意外再把它放回它所屬的 `Container`。這樣做的結果是，我們能連續地呼叫 `map`，執行任何我們想執行的函式。甚至還可以改變值的型別，就像上面最後一個例子中那樣。

等等，如果我們能一直呼叫 `map`，那它不就是個組合（composition）麼！這裡邊是有什麼數學魔法在起作用？是 *functor*。各位，這個數學魔法就是 *functor*。

> functor 是實現了 `map` 函式並遵守一些特定規則的容器型別。

沒錯，*functor* 就是一個簽了合約的介面。我們本來可以簡單地把它稱為 `Mappable`，但現在為時已晚，哪怕 *functor* 一點也不 *fun*。functor 是範疇學裡的概念，我們將在本章末尾詳細探索與此相關的數學知識；暫時我們先用這個名字很奇怪的介面做一些不那麼理論的、實用性的練習。

把值裝進一個容器，而且只能使用 `map` 來處理它，這麼做的理由到底是什麼呢？如果我們換種方式來問，答案就很明顯了：讓容器自己去運用函式能給我們帶來什麼好處？答案是抽象，對於函式運用的抽象。當 `map` 一個函式的時候，我們請求容器來執行這個函式。不誇張地講，這是一種十分強大的理念。

## 薛定諤的 Maybe

<img src="images/cat.png" alt="cool cat, need reference" />

說實話 `Container` 挺無聊的，而且通常我們稱它為 `Identity`，與 `id` 函式的作用相同（這裡也是有數學上的聯絡的，我們會在適當時候加以說明）。除此之外，還有另外一種 functor，那就是實現了 `map` 函式的類似容器的資料型別，這種 functor 在呼叫 `map` 的時候能夠提供非常有用的行為。現在讓我們來定義一個這樣的 functor。

```js
var Maybe = function(x) {
  this.__value = x;
}

Maybe.of = function(x) {
  return new Maybe(x);
}

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
}

Maybe.prototype.map = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
}
```

`Maybe` 看起來跟 `Container` 非常類似，但是有一點不同：`Maybe` 會先檢查自己的值是否為空，然後才呼叫傳進來的函式。這樣我們在使用 `map` 的時候就能避免惱人的空值了（注意這個實現出於教學目的做了簡化）。

```js
Maybe.of("Malkovich Malkovich").map(match(/a/ig));
//=> Maybe(['a', 'a'])

Maybe.of(null).map(match(/a/ig));
//=> Maybe(null)

Maybe.of({name: "Boris"}).map(_.prop("age")).map(add(10));
//=> Maybe(null)

Maybe.of({name: "Dinah", age: 14}).map(_.prop("age")).map(add(10));
//=> Maybe(24)
```

注意看，當傳給 `map` 的值是 `null` 時，程式碼並沒有爆出錯誤。這是因為每一次 `Maybe` 要呼叫函式的時候，都會先檢查它自己的值是否為空。

這種點記法（dot notation syntax）已經足夠函數式了，但是正如在第 1 部分指出的那樣，我們更想保持一種 pointfree 的風格。碰巧的是，`map` 完全有能力以 curry 函式的方式來“代理”任何 functor：

```js
//  map :: Functor f => (a -> b) -> f a -> f b
var map = curry(function(f, any_functor_at_all) {
  return any_functor_at_all.map(f);
});
```

這樣我們就可以像平常一樣使用組合，同時也能正常使用 `map` 了，非常振奮人心。ramda 的 `map` 也是這樣。後面的章節中，我們將在點記法更有教育意義的時候使用點記法，在方便使用 pointfree 模式的時候就用 pointfree。你注意到了麼？我在型別標籤中偷偷引入了一個額外的標記：`Functor f =>`。這個標記告訴我們 `f` 必須是一個 functor。沒什麼複雜的，但我覺得有必要提一下。

## 用例

實際當中，`Maybe` 最常用在那些可能會無法成功返回結果的函式中。

```js
//  safeHead :: [a] -> Maybe(a)
var safeHead = function(xs) {
  return Maybe.of(xs[0]);
};

var streetName = compose(map(_.prop('street')), safeHead, _.prop('addresses'));

streetName({addresses: []});
// Maybe(null)

streetName({addresses: [{street: "Shady Ln.", number: 4201}]});
// Maybe("Shady Ln.")
```

`safeHead` 與一般的 `_.head` 類似，但是增加了型別安全保證。引入 `Maybe` 會發生一件非常有意思的事情，那就是我們被迫要與狡猾的 `null` 打交道了。`safeHead` 函式能夠誠實地預告它可能的失敗——失敗真沒什麼可恥的——然後返回一個 `Maybe` 來通知我們相關資訊。實際上不僅僅是*通知*，因為畢竟我們想要的值深藏在 `Maybe` 物件中，而且只能通過 `map` 來操作它。本質上，這是一種由 `safeHead` 強制執行的空值檢查。有了這種檢查，我們才能在夜裡安然入睡，因為我們知道最不受人待見的 `null` 不會突然出現。類似這樣的 API 能夠把一個像紙糊起來的、脆弱的應用升級為實實在在的、健壯的應用，這樣的 API 保證了更加安全的軟體。

有時候函式可以明確返回一個 `Maybe(null)` 來表明失敗，例如：

```js
//  withdraw :: Number -> Account -> Maybe(Account)
var withdraw = curry(function(amount, account) {
  return account.balance >= amount ?
    Maybe.of({balance: account.balance - amount}) :
    Maybe.of(null);
});

//  finishTransaction :: Account -> String
var finishTransaction = compose(remainingBalance, updateLedger); // <- 假定這兩個函式已經在別處定義好了

//  getTwenty :: Account -> Maybe(String)
var getTwenty = compose(map(finishTransaction), withdraw(20));


getTwenty({ balance: 200.00});
// Maybe("Your balance is $180.00")

getTwenty({ balance: 10.00});
// Maybe(null)
```

要是錢不夠，`withdraw` 就會對我們嗤之以鼻然後返回一個 `Maybe(null)`。`withdraw` 也顯示出了它的多變性，使得我們後續的操作只能用 `map` 來進行。這個例子與前面例子不同的地方在於，這裡的 `null` 是有意的。我們不用 `Maybe(String)` ，而是用 `Maybe(null)` 來發送失敗的訊號，這樣程式在收到訊號後就能立刻停止執行。這一點很重要：如果 `withdraw` 失敗了，`map` 就會切斷後續程式碼的執行，因為它根本就不會執行傳遞給它的函式，即 `finishTransaction`。這正是預期的效果：如果取款失敗，我們並不想更新或者顯示賬戶餘額。

## 釋放容器裡的值

人們經常忽略的一個事實是：任何事物都有個最終盡頭。那些會產生作用的函式，不管它們是傳送 JSON 資料，還是在螢幕上列印東西，還是更改檔案系統，還是別的什麼，都要有一個結束。但是我們無法通過 `return` 把輸出傳遞到外部世界，必須要執行這樣或那樣的函式才能傳遞出去。關於這一點，可以借用禪宗公案的口吻來敘述：“如果一個程式執行之後沒有可觀察到的作用，那它到底運行了沒有？”。或者，執行之後達到自身的目的了沒有？有可能它只是浪費了幾個 CPU 週期然後就去睡覺了...

應用程式所做的工作就是獲取、更改和儲存資料直到不再需要它們，對資料做這些操作的函式有可能被 `map` 呼叫，這樣的話資料就可以不用離開它溫暖舒適的容器。諷刺的是，有一種常見的錯誤就是試圖以各種方法刪除 `Maybe` 裡的值，好像這個不確定的值是魔鬼，刪除它就能讓它突然顯形，然後一切罪惡都會得到寬恕似的（譯者注：此處原文應該是源自聖經）。要知道，我們的值沒有完成它的使命，很有可能是其他程式碼分支造成的。我們的程式碼，就像薛定諤的貓一樣，在某個特定的時間點有兩種狀態，而且應該保持這種狀況不變直到最後一個函式為止。這樣，哪怕程式碼有很多邏輯性的分支，也能保證一種線性的工作流。

不過，對容器裡的值來說，還是有個逃生口可以出去。也就是說，如果我們想返回一個自定義的值然後還能繼續執行後面的程式碼的話，是可以做到的；要達到這一目的，可以藉助一個幫助函式 `maybe`：

```js
//  maybe :: b -> (a -> b) -> Maybe a -> b
var maybe = curry(function(x, f, m) {
  return m.isNothing() ? x : f(m.__value);
});

//  getTwenty :: Account -> String
var getTwenty = compose(
  maybe("You're broke!", finishTransaction), withdraw(20)
);


getTwenty({ balance: 200.00});
// "Your balance is $180.00"

getTwenty({ balance: 10.00});
// "You're broke!"
```

這樣就可以要麼返回一個靜態值（與 `finishTransaction` 返回值的型別一致），要麼繼續愉快地在沒有 `Maybe` 的情況下完成交易。`maybe` 使我們得以避免普通 `map` 那種命令式的 `if/else` 語句：`if(x !== null) { return f(x) }`。

引入 `Maybe` 可能會在初期造成一些不適。Swift 和 Scala 使用者知道我在說什麼，因為這兩門語言的核心庫裡就有 `Maybe` 的概念，只不過偽裝成 `Option(al)` 罷了。被迫在任何情況下都進行空值檢查（甚至有些時候我們可以確定某個值不會為空），的確讓大部分人頭疼不已。然而隨著時間推移，空值檢查會成為第二本能，說不定你還會感激它提供的安全性呢。不管怎麼說，空值檢查大多數時候都能防止在程式碼邏輯上偷工減料，讓我們脫離危險。

編寫不安全的軟體就像用蠟筆小心翼翼地畫彩蛋，畫完之後把它們扔到大街上一樣（譯者注：意思是彩蛋非常易於尋找。來源於復活節習俗，人們會藏起一些彩蛋讓孩子尋找），或者像用三隻小豬警告過的材料蓋個養老院一樣（譯者注：來源於“三隻小豬”童話故事）。`Maybe` 能夠非常有效地幫助我們增加函式的安全性。

有一點我必須要提及，否則就太不負責任了，那就是 `Maybe` 的“真正”實現會把它分為兩種型別：一種是非空值，另一種是空值。這種實現允許我們遵守 `map` 的 parametricity 特性，因此 `null` 和 `undefined` 能夠依然被 `map` 呼叫，functor 裡的值所需的那種普遍性條件也能得到滿足。所以你會經常看到 `Some(x) / None` 或者 `Just(x) / Nothing` 這樣的容器型別在做空值檢查，而不是 `Maybe`。

## “純”錯誤處理

<img src="images/fists.jpg" alt="pick a hand... need a reference" />

說出來可能會讓你震驚，`throw/catch` 並不十分“純”。當一個錯誤丟擲的時候，我們沒有收到返回值，反而是得到了一個警告！拋錯的函式吐出一大堆的 0 和 1 作為盾和矛來攻擊我們，簡直就像是在反擊輸入值的入侵而進行的一場電子大作戰。有了 `Either` 這個新朋友，我們就能以一種比向輸入值宣戰好得多的方式來處理錯誤，那就是返回一條非常禮貌的訊息作為迴應。我們來看一下：

```js
var Left = function(x) {
  this.__value = x;
}

Left.of = function(x) {
  return new Left(x);
}

Left.prototype.map = function(f) {
  return this;
}

var Right = function(x) {
  this.__value = x;
}

Right.of = function(x) {
  return new Right(x);
}

Right.prototype.map = function(f) {
  return Right.of(f(this.__value));
}
```

`Left` 和 `Right` 是我們稱之為 `Either` 的抽象型別的兩個子類。我略去了建立 `Either` 父類的繁文縟節，因為我們不會用到它的，但你瞭解一下也沒壞處。注意看，這裡除了有兩個型別，沒別的新鮮東西。來看看它們是怎麼執行的：

```js
Right.of("rain").map(function(str){ return "b"+str; });
// Right("brain")

Left.of("rain").map(function(str){ return "b"+str; });
// Left("rain")

Right.of({host: 'localhost', port: 80}).map(_.prop('host'));
// Right('localhost')

Left.of("rolls eyes...").map(_.prop("host"));
// Left('rolls eyes...')
```

`Left` 就像是青春期少年那樣無視我們要 `map` 它的請求。`Right` 的作用就像是一個 `Container`（也就是 Identity）。這裡強大的地方在於，`Left` 有能力在它內部嵌入一個錯誤訊息。

假設有一個可能會失敗的函式，就拿根據生日計算年齡來說好了。的確，我們可以用 `Maybe(null)` 來表示失敗並把程式引向另一個分支，但是這並沒有告訴我們太多資訊。很有可能我們想知道失敗的原因是什麼。用 `Either` 寫一個這樣的程式看看：

```js
var moment = require('moment');

//  getAge :: Date -> User -> Either(String, Number)
var getAge = curry(function(now, user) {
  var birthdate = moment(user.birthdate, 'YYYY-MM-DD');
  if(!birthdate.isValid()) return Left.of("Birth date could not be parsed");
  return Right.of(now.diff(birthdate, 'years'));
});

getAge(moment(), {birthdate: '2005-12-12'});
// Right(9)

getAge(moment(), {birthdate: '20010704'});
// Left("Birth date could not be parsed")
```

這麼一來，就像 `Maybe(null)`，當返回一個 `Left` 的時候就直接讓程式短路。跟 `Maybe(null)` 不同的是，現在我們對程式為何脫離原先軌道至少有了一點頭緒。有一件事要注意，這裡返回的是 `Either(String, Number)`，意味著我們這個 `Either` 左邊的值是 `String`，右邊（譯者注：也就是正確的值）的值是 `Number`。這個型別簽名不是很正式，因為我們並沒有定義一個真正的 `Either` 父類；但我們還是從這個型別那裡瞭解到不少東西。它告訴我們，我們得到的要麼是一條錯誤訊息，要麼就是正確的年齡值。

```js
//  fortune :: Number -> String
var fortune  = compose(concat("If you survive, you will be "), add(1));

//  zoltar :: User -> Either(String, _)
var zoltar = compose(map(console.log), map(fortune), getAge(moment()));

zoltar({birthdate: '2005-12-12'});
// "If you survive, you will be 10"
// Right(undefined)

zoltar({birthdate: 'balloons!'});
// Left("Birth date could not be parsed")
```

如果 `birthdate` 合法，這個程式就會把它神祕的命運列印在螢幕上讓我們見證；如果不合法，我們就會收到一個有著清清楚楚的錯誤訊息的 `Left`，儘管這個訊息是穩穩當當地待在它的容器裡的。這種行為就像，雖然我們在拋錯，但是是以一種平靜溫和的方式拋錯，而不是像一個小孩子那樣，有什麼不對勁就鬧脾氣大喊大叫。

在這個例子中，我們根據 `birthdate` 的合法性來控制程式碼的邏輯分支，同時又讓程式碼進行從右到左的直線運動，而不用爬過各種條件語句的大括號。通常，我們不會把 `console.log` 放到 `zoltar` 函式裡，而是在呼叫 `zoltar` 的時候才 `map` 它，不過本例中，讓你看看 `Right` 分支如何與 `Left` 不同也是很有幫助的。我們在 `Right` 分支的型別簽名中使用 `_` 表示一個應該忽略的值（在有些瀏覽器中，你必須要 `console.log.bind(console)` 才能把 `console.log` 當作一等公民使用）。

我想借此機會指出一件你可能沒注意到的事：這個例子中，儘管 `fortune` 使用了 `Either`，它對每一個 functor 到底要幹什麼卻是毫不知情的。前面例子中的 `finishTransaction` 也是一樣。通俗點來講，一個函式在呼叫的時候，如果被 `map` 包裹了，那麼它就會從一個非 functor 函式轉換為一個 functor 函式。我們把這個過程叫做 *lift*。一般情況下，普通函式更適合操作普通的資料型別而不是容器型別，在必要的時候再通過 *lift* 變為合適的容器去操作容器型別。這樣做的好處是能得到更簡單、重用性更高的函式，它們能夠隨需求而變，相容任意 functor。

`Either` 並不僅僅只對合法性檢查這種一般性的錯誤作用非凡，對一些更嚴重的、能夠中斷程式執行的錯誤比如檔案丟失或者 socket 連線斷開等，`Either` 同樣效果顯著。你可以試試把前面例子中的 `Maybe` 替換為 `Either`，看怎麼得到更好的反饋。

此刻我忍不住在想，我僅僅是把 `Either` 當作一個錯誤訊息的容器介紹給你！這樣的介紹有失偏頗，它的能耐遠不止於此。比如，它表示了邏輯或（也就是 `||`）。再比如，它體現了範疇學裡 *coproduct* 的概念，當然本書不會涉及這方面的知識，但值得你去深入瞭解，因為這個概念有很多特性值得利用。還比如，它是標準的 sum type（或者叫不交併集，disjoint union of sets），因為它含有的所有可能的值的總數就是它包含的那兩種型別的總數（我知道這麼說你聽不懂，沒關係，這裡有一篇[非常棒的文章](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/sum-types)講述這個問題）。`Either` 能做的事情多著呢，但是作為一個 functor，我們就用它處理錯誤。

就像 `Maybe` 可以有個 `maybe` 一樣，`Either` 也可以有一個 `either`。兩者的用法類似，但 `either` 接受兩個函式（而不是一個）和一個靜態值為引數。這兩個函式的返回值型別一致：

```js
//  either :: (a -> c) -> (b -> c) -> Either a b -> c
var either = curry(function(f, g, e) {
  switch(e.constructor) {
    case Left: return f(e.__value);
    case Right: return g(e.__value);
  }
});

//  zoltar :: User -> _
var zoltar = compose(console.log, either(id, fortune), getAge(moment()));

zoltar({birthdate: '2005-12-12'});
// "If you survive, you will be 10"
// undefined

zoltar({birthdate: 'balloons!'});
// "Birth date could not be parsed"
// undefined
```

終於用了一回那個神祕的 `id` 函式！其實它就是簡單地複製了 `Left` 裡的錯誤訊息，然後把這個值傳給 `console.log` 而已。通過強制在 `getAge` 內部進行錯誤處理，我們的算命程式更加健壯了。結果就是，要麼告訴使用者一個殘酷的事實並像算命師那樣跟他擊掌，要麼就繼續執行程式。好了，現在我們已經準備好去學習一個完全不同型別的 functor 了。

## 王老先生有作用...

（譯者注：原標題是“Old McDonald had Effects...”，源於美國兒歌“Old McDonald Had a Farm”。）

<img src="images/dominoes.jpg" alt="dominoes.. need a reference" />

在關於純函式的的那一章（即第 3 章）裡，有一個很奇怪的例子。這個例子中的函式會產生副作用，但是我們通過把它包裹在另一個函式裡的方式把它變得看起來像一個純函式。這裡還有一個類似的例子：

```js
//  getFromStorage :: String -> (_ -> String)
var getFromStorage = function(key) {
  return function() {
    return localStorage[key];
  }
}
```

要是我們沒把 `getFromStorage` 包在另一個函式裡，它的輸出值就是不定的，會隨外部環境變化而變化。有了這個結實的包裹函式（wrapper），同一個輸入就總能返回同一個輸出：一個從 `localStorage` 裡取出某個特定的元素的函式。就這樣（也許再高唱幾句讚美聖母的讚歌）我們洗滌了心靈，一切都得到了寬恕。

然而，這並沒有多大的用處，你說是不是。就像是你收藏的全新未拆封的玩偶，不能拿出來玩有什麼意思。所以要是能有辦法進到這個容器裡面，拿到它藏在那兒的東西就好了...辦法是有的，請看 `IO`：

```js
var IO = function(f) {
  this.__value = f;
}

IO.of = function(x) {
  return new IO(function() {
    return x;
  });
}

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.__value));
}
```

`IO` 跟之前的 functor 不同的地方在於，它的 `__value` 總是一個函式。不過我們不把它當作一個函式——實現的細節我們最好先不管。這裡發生的事情跟我們在 `getFromStorage` 那裡看到的一模一樣：`IO` 把非純執行動作（impure action）捕獲到包裹函式裡，目的是延遲執行這個非純動作。就這一點而言，我們認為 `IO` 包含的是被包裹的執行動作的返回值，而不是包裹函式本身。這在 `of` 函式裡很明顯：`IO(function(){ return x })` 僅僅是為了延遲執行，其實我們得到的是 `IO(x)`。

來用用看：

```js
//  io_window_ :: IO Window
var io_window = new IO(function(){ return window; });

io_window.map(function(win){ return win.innerWidth });
// IO(1430)

io_window.map(_.prop('location')).map(_.prop('href')).map(split('/'));
// IO(["http:", "", "localhost:8000", "blog", "posts"])


//  $ :: String -> IO [DOM]
var $ = function(selector) {
  return new IO(function(){ return document.querySelectorAll(selector); });
}

$('#myDiv').map(head).map(function(div){ return div.innerHTML; });
// IO('I am some inner html')
```

這裡，`io_window` 是一個真正的 `IO`，我們可以直接對它使用 `map`。至於 `$`，則是一個函式，呼叫後會返回一個 `IO`。我把這裡的返回值都寫成了*概念性*的，這樣就更加直觀；不過實際的返回值是 `{ __value: [Function] }`。當呼叫 `IO` 的 `map` 的時候，我們把傳進來的函式放在了 `map` 函式裡的組合的最末端（也就是最左邊），反過來這個函式就成為了新的 `IO` 的新 `__value`，並繼續下去。傳給 `map` 的函式並沒有執行，我們只是把它們壓到一個“執行棧”的最末端而已，一個函式緊挨著另一個函式，就像小心擺放的多米諾骨牌一樣，讓人不敢輕易推倒。這種情形很容易叫人聯想起“四人幫”（譯者注：《設計模式》一書作者）提出的命令模式（command pattern）或者佇列（queue）。

花點時間找回你關於 functor 的直覺吧。把實現細節放在一邊不管，你應該就能自然而然地對各種各樣的容器使用 `map` 了，不管它是多麼奇特怪異。這種偽超自然的力量要歸功於 functor 的定律，我們將在本章末尾對此作一番探索。無論如何，我們終於可以在不犧牲程式碼純粹性的情況下，隨意使用這些不純的值了。

好了，我們已經把野獸關進了籠子。但是，在某一時刻還是要把它放出來。因為對 `IO` 呼叫 `map` 已經積累了太多不純的操作，最後再執行它無疑會打破平靜。問題是在哪裡，什麼時候開啟籠子的開關？而且有沒有可能我們只執行 `IO` 卻不讓不純的操作弄髒雙手？答案是可以的，只要把責任推到呼叫者身上就行了。我們的純程式碼，儘管陰險狡詐詭計多端，但是卻始終保持一副清白無辜的模樣，反而是實際執行 `IO` 併產生了作用的呼叫者，背了黑鍋。來看一個具體的例子。

```js

////// 純程式碼庫: lib/params.js ///////

//  url :: IO String
var url = new IO(function() { return window.location.href; });

//  toPairs =  String -> [[String]]
var toPairs = compose(map(split('=')), split('&'));

//  params :: String -> [[String]]
var params = compose(toPairs, last, split('?'));

//  findParam :: String -> IO Maybe [String]
var findParam = function(key) {
  return map(compose(Maybe.of, filter(compose(eq(key), head)), params), url);
};

////// 非純呼叫程式碼: main.js ///////

// 呼叫 __value() 來執行它！
findParam("searchTerm").__value();
// Maybe(['searchTerm', 'wafflehouse'])
```

lib/params.js 把 `url` 包裹在一個 `IO` 裡，然後把這頭野獸傳給了呼叫者；一雙手保持的非常乾淨。你可能也注意到了，我們把容器也“壓棧”了，要知道建立一個 `IO(Maybe([x]))` 沒有任何不合理的地方。我們這個“棧”有三層 functor（`Array` 是最有資格成為 mappable 的容器型別），令人印象深刻。

有件事困擾我很久了，現在我必須得說出來：`IO` 的 `__value` 並不是它包含的值，也不是像兩個下劃線暗示那樣是一個私有屬性。`__value` 是手榴彈的彈栓，只應該被呼叫者以最公開的方式拉動。為了提醒使用者它的變化無常，我們把它重新命名為 `unsafePerformIO` 看看。

```js
var IO = function(f) {
  this.unsafePerformIO = f;
}

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.unsafePerformIO));
}
```

看，這就好多了。現在呼叫的程式碼就變成了 `findParam("searchTerm").unsafePerformIO()`，對應用程式的使用者（以及本書讀者）來說，這簡直就直白得不能再直白了。

`IO` 會成為一個忠誠的伴侶，幫助我們馴化那些狂野的非純操作。下一節我們將學習一種跟 `IO` 在精神上相似，但是用法上又千差萬別的型別。


## 非同步任務

回撥（callback）是通往地獄的狹窄的螺旋階梯。它們是埃舍爾（譯者注：荷蘭版畫藝術家）設計的控制流。看到一個個巢狀的回撥擠在大小括號搭成的架子上，讓人不由自主地聯想到地牢裡的靈薄獄（還能再低點麼！）（譯者注：靈薄獄即 limbo，基督教中地獄邊緣之意）。光是想到這樣的回撥就讓我幽閉恐怖症發作了。不過別擔心，處理非同步程式碼，我們有一種更好的方式，它的名字以“F”開頭。

這種方式的內部機制過於複雜，複雜得哪怕我唾沫橫飛也很難講清楚。所以我們就直接用 Quildreen Motta 的 [Folktale](http://folktalejs.org/) 裡的 `Data.Task` （之前是 `Data.Future`）。來見證一些例子吧：

```js
// Node readfile example:
//=======================

var fs = require('fs');

//  readFile :: String -> Task(Error, JSON)
var readFile = function(filename) {
  return new Task(function(reject, result) {
    fs.readFile(filename, 'utf-8', function(err, data) {
      err ? reject(err) : result(data);
    });
  });
};

readFile("metamorphosis").map(split('\n')).map(head);
// Task("One morning, as Gregor Samsa was waking up from anxious dreams, he discovered that
// in bed he had been changed into a monstrous verminous bug.")


// jQuery getJSON example:
//========================

//  getJSON :: String -> {} -> Task(Error, JSON)
var getJSON = curry(function(url, params) {
  return new Task(function(reject, result) {
    $.getJSON(url, params, result).fail(reject);
  });
});

getJSON('/video', {id: 10}).map(_.prop('title'));
// Task("Family Matters ep 15")

// 傳入普通的實際值也沒問題
Task.of(3).map(function(three){ return three + 1 });
// Task(4)
```

例子中的 `reject` 和 `result` 函式分別是失敗和成功的回撥。正如你看到的，我們只是簡單地呼叫 `Task` 的 `map` 函式，就能操作將來的值，好像這個值就在那兒似的。到現在 `map` 對你來說應該不稀奇了。

如果熟悉 promise 的話，你該能認出來 `map` 就是 `then`，`Task` 就是一個 promise。如果不熟悉你也不必氣餒，反正我們也不會用它，因為它並不純；但剛才的類比還是成立的。

與 `IO` 類似，`Task` 在我們給它綠燈之前是不會執行的。事實上，正因為它要等我們的命令，`IO` 實際就被納入到了 `Task` 名下，代表所有的非同步操作——`readFile` 和 `getJSON` 並不需要一個額外的 `IO` 容器來變純。更重要的是，當我們呼叫它的 `map` 的時候，`Task` 工作的方式與 `IO` 幾無差別：都是把對未來的操作的指示放在一個時間膠囊裡，就像家務列表（chore chart）那樣——真是一種精密的拖延術。

我們必須呼叫 `fork` 方法才能執行 `Task`，這種機制與 `unsafePerformIO` 類似。但也有不同，不同之處就像 `fork` 這個名稱表明的那樣，它會 fork 一個子程序執行它接收到的引數程式碼，其他部分的執行不受影響，主執行緒也不會阻塞。當然這種效果也可以用其他一些技術比如執行緒實現，但這裡的這種方法工作起來就像是一個普通的非同步呼叫，而且 event loop 能夠不受影響地繼續運轉。我們來看一下 `fork`：

```js
// Pure application
//=====================
// blogTemplate :: String

//  blogPage :: Posts -> HTML
var blogPage = Handlebars.compile(blogTemplate);

//  renderPage :: Posts -> HTML
var renderPage = compose(blogPage, sortBy('date'));

//  blog :: Params -> Task(Error, HTML)
var blog = compose(map(renderPage), getJSON('/posts'));


// Impure calling code
//=====================
blog({}).fork(
  function(error){ $("#error").html(error.message); },
  function(page){ $("#main").html(page); }
);

$('#spinner').show();
```

呼叫 `fork` 之後，`Task` 就趕緊跑去找一些文章，渲染到頁面上。與此同時，我們在頁面上展示一個 spinner，因為 `fork` 不會等收到響應了才執行它後面的程式碼。最後，我們要麼把文章展示在頁面上，要麼就顯示一個出錯資訊，視 `getJSON` 請求是否成功而定。

花點時間思考下這裡的控制流為何是線性的。我們只需要從下讀到上，從右讀到左就能理解程式碼，即便這段程式實際上會在執行過程中到處跳來跳去。這種方式使得閱讀和理解應用程式的程式碼比那種要在各種回撥和錯誤處理程式碼塊之間跳躍的方式容易得多。

天哪，你看到了麼，`Task` 居然也包含了 `Either`！沒辦法，為了能處理將來可能出現的錯誤，它必須得這麼做，因為普通的控制流在非同步的世界裡不適用。這自然是好事一樁，因為它天然地提供了充分的“純”錯誤處理。

就算是有了 `Task`，`IO` 和 `Either` 這兩個 functor 也照樣能派上用場。待我舉個簡單例子向你說明一種更復雜、更假想的情況，雖然如此，這個例子還是能夠說明我的目的。

```js
// Postgres.connect :: Url -> IO DbConnection
// runQuery :: DbConnection -> ResultSet
// readFile :: String -> Task Error String

// Pure application
//=====================

//  dbUrl :: Config -> Either Error Url
var dbUrl = function(c) {
  return (c.uname && c.pass && c.host && c.db)
    ? Right.of("db:pg://"+c.uname+":"+c.pass+"@"+c.host+"5432/"+c.db)
    : Left.of(Error("Invalid config!"));
}

//  connectDb :: Config -> Either Error (IO DbConnection)
var connectDb = compose(map(Postgres.connect), dbUrl);

//  getConfig :: Filename -> Task Error (Either Error (IO DbConnection))
var getConfig = compose(map(compose(connectDB, JSON.parse)), readFile);


// Impure calling code
//=====================
getConfig("db.json").fork(
  logErr("couldn't read file"), either(console.log, map(runQuery))
);
```

這個例子中，我們在 `readFile` 成功的那個程式碼分支裡利用了 `Either` 和 `IO`。`Task` 處理非同步讀取檔案這一操作當中的不“純”性，但是驗證 config 的合法性以及連線資料庫則分別使用了 `Either` 和 `IO`。所以你看，我們依然在同步地跟所有事物打交道。

例子我還可以再舉一些，但是就到此為止吧。這些概念就像 `map` 一樣簡單。

實際當中，你很有可能在一個工作流中跑好幾個非同步任務，但我們還沒有完整學習容器的 api 來應對這種情況。不必擔心，我們很快就會去學習 monad 之類的概念。不過，在那之前，我們得先檢查下所有這些背後的數學知識。


## 一點理論

前面提到，functor 的概念來自於範疇學，並滿足一些定律。我們先來探索這些實用的定律。

```js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

*同一律*很簡單，但是也很重要。因為這些定律都是可執行的程式碼，所以我們完全可以在我們自己的 functor 上試驗它們，驗證它們是否成立。

```js
var idLaw1 = map(id);
var idLaw2 = id;

idLaw1(Container.of(2));
//=> Container(2)

idLaw2(Container.of(2));
//=> Container(2)
```

看到沒，它們是相等的。接下來看一看組合。

```js
var compLaw1 = compose(map(concat(" world")), map(concat(" cruel")));
var compLaw2 = map(compose(concat(" world"), concat(" cruel")));

compLaw1(Container.of("Goodbye"));
//=> Container('Goodbye cruel world')

compLaw2(Container.of("Goodbye"));
//=> Container('Goodbye cruel world')
```

在範疇學中，functor 接受一個範疇的物件和態射（morphism），然後把它們對映（map）到另一個範疇裡去。根據定義，這個新範疇一定會有一個單位元（identity），也一定能夠組合態射；我們無須驗證這一點，前面提到的定律保證這些東西會在對映後得到保留。

可能我們關於範疇的定義還是有點模糊。你可以把範疇想象成一個有著多個物件的網路，物件之間靠態射連線。那麼 functor 可以把一個範疇對映到另外一個，而且不會破壞原有的網路。如果一個物件 `a` 屬於源範疇 `C`，那麼通過 functor `F` 把 `a` 對映到目標範疇 `D` 上之後，就可以使用 `F a` 來指代 `a` 物件（把這些字母拼起來是什麼？！）。可能看圖會更容易理解：

<img src="images/catmap.png" alt="Categories mapped" />

比如，`Maybe` 就把型別和函式的範疇對映到這樣一個範疇：即每個物件都有可能不存在，每個態射都有空值檢查的範疇。這個結果在程式碼中的實現方式是用 `map` 包裹每一個函式，用 functor 包裹每一個型別。這樣就能保證每個普通的型別和函式都能在新環境下繼續使用組合。從技術上講，程式碼中的 functor 實際上是把範疇對映到了一個包含型別和函式的子範疇（sub category）上，使得這些 functor 成為了一種新的特殊的 endofunctor。但出於本書的目的，我們認為它就是一個不同的範疇。

可以用一張圖來表示這種態射及其物件的對映：

<img src="images/functormap.png" alt="functor diagram" />

這張圖除了能表示態射藉助 functor `F` 完成從一個範疇到另一個範疇的對映之外，我們發現它還符合交換律，也就是說，順著箭頭的方向往前，形成的每一個路徑都指向同一個結果。不同的路徑意味著不同的行為，但最終都會得到同一個資料型別。這種形式化給了我們原則性的方式去思考程式碼——無須分析和評估每一個單獨的場景，只管可以大膽地應用公式即可。來看一個具體的例子。

```js
//  topRoute :: String -> Maybe(String)
var topRoute = compose(Maybe.of, reverse);

//  bottomRoute :: String -> Maybe(String)
var bottomRoute = compose(map(reverse), Maybe.of);


topRoute("hi");
// Maybe("ih")

bottomRoute("hi");
// Maybe("ih")
```

或者看圖：

<img src="images/functormapmaybe.png" alt="functor diagram 2" />

根據所有 functor 都有的特性，我們可以立即理解程式碼，重構程式碼。

functor 也能巢狀使用：

```js
var nested = Task.of([Right.of("pillows"), Left.of("no sleep for you")]);

map(map(map(toUpperCase)), nested);
// Task([Right("PILLOWS"), Left("no sleep for you")])
```

`nested` 是一個將來的陣列，陣列的元素有可能是程式丟擲的錯誤。我們使用 `map` 剝開每一層的巢狀，然後對陣列的元素呼叫傳遞進去的函式。可以看到，這中間沒有回撥、`if/else` 語句和 `for` 迴圈，只有一個明確的上下文。的確，我們必須要 `map(map(map(f)))` 才能最終執行函式。不想這麼做的話，可以組合 functor。是的，你沒聽錯：

```js
var Compose = function(f_g_x){
  this.getCompose = f_g_x;
}

Compose.prototype.map = function(f){
  return new Compose(map(map(f), this.getCompose));
}

var tmd = Task.of(Maybe.of("Rock over London"))

var ctmd = new Compose(tmd);

map(concat(", rock on, Chicago"), ctmd);
// Compose(Task(Maybe("Rock over London, rock on, Chicago")))

ctmd.getCompose;
// Task(Maybe("Rock over London, rock on, Chicago"))
```

看，只有一個 `map`。functor 組合是符合結合律的，而且之前我們定義的 `Container` 實際上是一個叫 `Identity` 的 functor。identity 和可結合的組合也能產生一個範疇，這個特殊的範疇的物件是其他範疇，態射是 functor。這實在太傷腦筋了，所以我們不會深入這個問題，但是讚歎一下這種模式的結構性含義，或者它的簡單的抽象之美也是好的。


## 總結

我們已經認識了幾個不同的 functor，但它們的數量其實是無限的。有一些值得注意的可迭代資料型別（iterable data structure）我們沒有介紹，像 tree、list、map 和 pair 等，以及所有你能說出來的。eventstream 和 observable 也都是 functor。其他的 functor 可能就是拿來做封裝或者僅僅是模擬型別。我們身邊到處都有 functor 的身影，本書也將會大量使用它們。

用多個 functor 引數呼叫一個函式怎麼樣呢？處理一個由不純的或者非同步的操作組成的有序序列怎麼樣呢？要應對這個什麼都裝在盒子裡的世界，目前我們工具箱裡的工具還不全。下一章，我們將直奔 monad 而去。

[第 9 章: Monad](ch9.md)

## 練習

```js
require('../../support');
var Task = require('data.task');
var _ = require('ramda');

// 練習 1
// ==========
// 使用 _.add(x,y) 和 _.map(f,x) 建立一個能讓 functor 裡的值增加的函式

var ex1 = undefined



//練習 2
// ==========
// 使用 _.head 獲取列表的第一個元素
var xs = Identity.of(['do', 'ray', 'me', 'fa', 'so', 'la', 'ti', 'do']);

var ex2 = undefined



// 練習 3
// ==========
// 使用 safeProp 和 _.head 找到 user 的名字的首字母
var safeProp = _.curry(function (x, o) { return Maybe.of(o[x]); });

var user = { id: 2, name: "Albert" };

var ex3 = undefined


// 練習 4
// ==========
// 使用 Maybe 重寫 ex4，不要有 if 語句

var ex4 = function (n) {
  if (n) { return parseInt(n); }
};

var ex4 = undefined



// 練習 5
// ==========
// 寫一個函式，先 getPost 獲取一篇文章，然後 toUpperCase 讓這片文章標題變為大寫

// getPost :: Int -> Future({id: Int, title: String})
var getPost = function (i) {
  return new Task(function(rej, res) {
    setTimeout(function(){
      res({id: i, title: 'Love them futures'})
    }, 300)
  });
}

var ex5 = undefined



// 練習 6
// ==========
// 寫一個函式，使用 checkActive() 和 showWelcome() 分別允許訪問或返回錯誤

var showWelcome = _.compose(_.add( "Welcome "), _.prop('name'))

var checkActive = function(user) {
 return user.active ? Right.of(user) : Left.of('Your account is not active')
}

var ex6 = undefined



// 練習 7
// ==========
// 寫一個驗證函式，檢查引數是否 length > 3。如果是就返回 Right(x)，否則就返回
// Left("You need > 3")

var ex7 = function(x) {
  return undefined // <--- write me. (don't be pointfree)
}



// 練習 8
// ==========
// 使用練習 7 的 ex7 和 Either 構造一個 functor，如果一個 user 合法就儲存它，否則
// 返回錯誤訊息。別忘了 either 的兩個引數必須返回同一型別的資料。

var save = function(x){
  return new IO(function(){
    console.log("SAVED USER!");
    return x + '-saved';
  });
}

var ex8 = undefined
```
