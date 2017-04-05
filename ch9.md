# Monad

## pointed functor

在繼續後面的內容之前，我得向你坦白一件事：關於我們先前建立的容器型別上的 `of` 方法，我並沒有說出它的全部實情。真實情況是，`of` 方法不是用來避免使用 `new` 關鍵字的，而是用來把值放到*預設最小化上下文*（default minimal context）中的。是的，`of` 沒有真正地取代構造器——它是一個我們稱之為 *pointed* 的重要介面的一部分。

> *pointed functor* 是實現了 `of` 方法的 functor。

這裡的關鍵是把任意值丟到容器裡然後開始到處使用 `map` 的能力。

```js
IO.of("tetris").map(concat(" master"));
// IO("tetris master")

Maybe.of(1336).map(add(1));
// Maybe(1337)

Task.of([{id: 2}, {id: 3}]).map(_.prop('id'));
// Task([2,3])

Either.of("The past, present and future walk into a bar...").map(
  concat("it was tense.")
);
// Right("The past, present and future walk into a bar...it was tense.")
```

如果你還記得，`IO` 和 `Task` 的構造器接受一個函式作為引數，而 `Maybe` 和 `Either` 的構造器可以接受任意值。實現這種介面的動機是，我們希望能有一種通用、一致的方式往 functor 裡填值，而且中間不會涉及到複雜性，也不會涉及到對構造器的特定要求。“預設最小化上下文”這個術語可能不夠精確，但是卻很好地傳達了這種理念：我們希望容器型別裡的任意值都能發生 `lift`，然後像所有的 functor 那樣再 `map` 出去。

有件很重要的事我必須得在這裡糾正，那就是，`Left.of` 沒有任何道理可言，包括它的雙關語也是。每個 functor 都要有一種把值放進去的方式，對 `Either` 來說，它的方式就是 `new Right(x)`。我們為 `Right` 定義 `of` 的原因是，如果一個型別容器*可以* `map`，那它就*應該* `map`。看上面的例子，你應該會對 `of` 通常的工作模式有一個直觀的印象，而 `Left` 破壞了這種模式。

你可能已經聽說過 `pure`、`point`、`unit` 和 `return` 之類的函數了，它們都是 `of` 這個史上最神祕函式的不同名稱（譯者注：此處原文是“international function of mystery”，源自惡搞《007》的電影 *Austin Powers: International Man of Mystery*，中譯名《王牌大賤諜》）。`of` 將在我們開始使用 monad 的時候顯示其重要性，因為後面你會看到，手動把值放回容器是我們自己的責任。

要避免 `new` 關鍵字，可以藉助一些標準的 JavaScript 技巧或者類庫達到目的。所以從這裡開始，我們就利用這些技巧或類庫，像一個負責任的成年人那樣使用 `of`。我推薦使用 `folktale`、`ramda` 或 `fantasy-land` 裡的 functor 例項，因為它們同時提供了正確的 `of` 方法和不依賴 `new` 的構造器。

## 混合比喻

<img src="images/onion.png" alt="http://www.organicchemistry.com/wp-content/uploads/BPOCchapter6-6htm-41.png" />

你看，除了太空墨西哥卷（如果你聽說過這個傳言的話）（譯者注：此處的傳言似乎是說一個叫 Chris Hadfield 的宇航員在國際空間站做墨西哥卷的事，[視訊連結](https://www.youtube.com/watch?v=f8-UKqGZ_hs)），monad 還被喻為洋蔥。讓我以一個常見的場景來說明這點：

```js
// Support
// ===========================
var fs = require('fs');

//  readFile :: String -> IO String
var readFile = function(filename) {
  return new IO(function() {
    return fs.readFileSync(filename, 'utf-8');
  });
};

//  print :: String -> IO String
var print = function(x) {
  return new IO(function() {
    console.log(x);
    return x;
  });
}

// Example
// ===========================
//  cat :: IO (IO String)
var cat = compose(map(print), readFile);

cat(".git/config")
// IO(IO("[core]\nrepositoryformatversion = 0\n"))
```

這裡我們得到的是一個 `IO`，只不過它陷進了另一個 `IO`。要想使用它，我們必須這樣呼叫： `map(map(f))`；要想觀察它的作用，必須這樣： `unsafePerformIO().unsafePerformIO()`。

```js
//  cat :: String -> IO (IO String)
var cat = compose(map(print), readFile);

//  catFirstChar :: String -> IO (IO String)
var catFirstChar = compose(map(map(head)), cat);

catFirstChar(".git/config")
// IO(IO("["))
```

儘管在應用中把這兩個作用打包在一起沒什麼不好的，但總感覺像是在穿著兩套防護服工作，結果就形成一個稀奇古怪的 API。再來看另一種情況：

```js
//  safeProp :: Key -> {Key: a} -> Maybe a
var safeProp = curry(function(x, obj) {
  return new Maybe(obj[x]);
});

//  safeHead :: [a] -> Maybe a
var safeHead = safeProp(0);

//  firstAddressStreet :: User -> Maybe (Maybe (Maybe Street) )
var firstAddressStreet = compose(
  map(map(safeProp('street'))), map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```

這裡的 functor 同樣是巢狀的，函式中三個可能的失敗都用了 `Maybe` 做預防也很乾淨整潔，但是要讓最後的呼叫者呼叫三次 `map` 才能取到值未免也太無禮了點——我們和它才剛剛見面而已。這種巢狀 functor 的模式會時不時地出現，而且是 monad 的主要使用場景。

我說過 monad 像洋蔥，那是因為當我們用 `map` 剝開巢狀的 functor 以獲取它裡面的值的時候，就像剝洋蔥一樣讓人忍不住想哭。不過，我們可以擦乾眼淚，做個深呼吸，然後使用一個叫作 `join` 的方法。

```js
var mmo = Maybe.of(Maybe.of("nunchucks"));
// Maybe(Maybe("nunchucks"))

mmo.join();
// Maybe("nunchucks")

var ioio = IO.of(IO.of("pizza"));
// IO(IO("pizza"))

ioio.join()
// IO("pizza")

var ttt = Task.of(Task.of(Task.of("sewers")));
// Task(Task(Task("sewers")));

ttt.join()
// Task(Task("sewers"))
```

如果有兩層相同型別的巢狀，那麼就可以用 `join` 把它們壓扁到一塊去。這種結合的能力，functor 之間的聯姻，就是 monad 之所以成為 monad 的原因。來看看它更精確的完整定義：

> monad 是可以變扁（flatten）的 pointed functor。

一個 functor，只要它定義個了一個 `join` 方法和一個 `of` 方法，並遵守一些定律，那麼它就是一個 monad。`join` 的實現並不太複雜，我們來為 `Maybe` 定義一個：

```js
Maybe.prototype.join = function() {
  return this.isNothing() ? Maybe.of(null) : this.__value;
}
```

看，就像子宮裡雙胞胎中的一個吃掉另一個那麼簡單。如果有一個 `Maybe(Maybe(x))`，那麼 `.__value` 將會移除多餘的一層，然後我們就能安心地從那開始進行 `map`。要不然，我們就將會只有一個 `Maybe`，因為從一開始就沒有任何東西被 `map` 呼叫。

既然已經有了 `join` 方法，我們把 monad 魔法作用到 `firstAddressStreet` 例子上，看看它的實際作用：

```js
//  join :: Monad m => m (m a) -> m a
var join = function(mma){ return mma.join(); }

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe({name: 'Mulburry', number: 8402})
```

只要遇到巢狀的 `Maybe`，就加一個 `join`，防止它們從手中溜走。我們對 `IO` 也這麼做試試看，感受下這種感覺。

```js
IO.prototype.join = function() {
  return this.unsafePerformIO();
}
```

同樣是簡單地移除了一層容器。注意，我們還沒有提及純粹性的問題，僅僅是移除過度緊縮的包裹中的一層而已。

```js
//  log :: a -> IO a
var log = function(x) {
  return new IO(function() { console.log(x); return x; });
}

//  setStyle :: Selector -> CSSProps -> IO DOM
var setStyle = curry(function(sel, props) {
  return new IO(function() { return jQuery(sel).css(props); });
});

//  getItem :: String -> IO String
var getItem = function(key) {
  return new IO(function() { return localStorage.getItem(key); });
};

//  applyPreferences :: String -> IO DOM
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);


applyPreferences('preferences').unsafePerformIO();
// Object {backgroundColor: "green"}
// <div style="background-color: 'green'"/>
```

`getItem` 返回了一個 `IO String`，所以可以直接用 `map` 來解析它。`log` 和 `setStyle` 返回的都是 `IO`，所以必須要使用 `join` 來保證這裡邊的巢狀處於控制之中。

## chain 函式

（譯者注：此處標題原文是“My chain hits my chest”，是英國歌手 M.I.A 單曲 *Bad Girls* 的一句歌詞。據說這首歌有體現女權主義。）

<img src="images/chain.jpg" alt="chain" />

你可能已經從上面的例子中注意到這種模式了：我們總是在緊跟著 `map` 的後面呼叫 `join`。讓我們把這個行為抽象到一個叫做 `chain` 的函式裡。

```js
//  chain :: Monad m => (a -> m b) -> m a -> m b
var chain = curry(function(f, m){
  return m.map(f).join(); // 或者 compose(join, map(f))(m)
});
```

這裡僅僅是把 map/join 套餐打包到一個單獨的函式中。如果你之前瞭解過 monad，那你可能已經看出來 `chain` 叫做 `>>=`（讀作 bind）或者 `flatMap`；都是同一個概念的不同名稱罷了。我個人認為 `flatMap` 是最準確的名稱，但本書還是堅持使用 `chain`，因為它是 JS 裡接受程度最高的一個。我們用 `chain` 重構下上面兩個例子：

```js
// map/join
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

// chain
var firstAddressStreet = compose(
  chain(safeProp('street')), chain(safeHead), safeProp('addresses')
);



// map/join
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);

// chain
var applyPreferences = compose(
  chain(setStyle), chain(log), map(JSON.parse), getItem
);
```

我把所有的 `map/join` 都替換為了 `chain`，這樣程式碼就顯得整潔了些。整潔固然是好事，但 `chain` 的能力卻不止於此——它更多的是龍捲風而不是吸塵器。因為 `chain` 可以輕鬆地巢狀多個作用，因此我們就能以一種純函式式的方式來表示 *序列*（sequence） 和 *變數賦值*（variable assignment）。

```js
// getJSON :: Url -> Params -> Task JSON
// querySelector :: Selector -> IO DOM


getJSON('/authenticate', {username: 'stale', password: 'crackers'})
  .chain(function(user) {
    return getJSON('/friends', {user_id: user.id});
});
// Task([{name: 'Seimith', id: 14}, {name: 'Ric', id: 39}]);


querySelector("input.username").chain(function(uname) {
  return querySelector("input.email").chain(function(email) {
    return IO.of(
      "Welcome " + uname.value + " " + "prepare for spam at " + email.value
    );
  });
});
// IO("Welcome Olivia prepare for spam at olivia@tremorcontrol.net");


Maybe.of(3).chain(function(three) {
  return Maybe.of(2).map(add(three));
});
// Maybe(5);


Maybe.of(null).chain(safeProp('address')).chain(safeProp('street'));
// Maybe(null);
```

本來我們可以用 `compose` 寫上面的例子，但這將需要幾個幫助函式，而且這種風格怎麼說都要通過閉包進行明確的變數賦值。相反，我們使用了插入式的 `chain`。順便說一下，`chain` 可以自動從任意型別的 `map` 和 `join` 衍生出來，就像這樣：`t.prototype.chain = function(f) { return this.map(f).join(); }`。如果手動定義 `chain` 能讓你覺得效能會好點的話（實際上並不會），我們也可以手動定義它，儘管還必須要費力保證函式功能的正確性——也就是說，它必須與緊接著後面有 `join` 的 `map` 相等。如果 `chain` 是簡單地通過結束呼叫 `of` 後把值放回容器這種方式定義的，那麼就會造成一個有趣的後果，即可以從 `chain` 那裡衍生出一個 `map`。同樣地，我們還可以用 `chain(id)` 定義 `join`。聽起來好像是在跟魔術師玩德州撲克，魔術師想要什麼牌就有什麼牌；但是就像大部分的數學理論一樣，所有這些原則性的結構都是相互關聯的。[fantasyland](https://github.com/fantasyland/fantasy-land) 倉庫中提到了許多上述衍生概念，這個倉庫也是 JavaScript 官方的代數資料結構（algebraic data types）標準。

好了，我們來看上面的例子。第一個例子中，可以看到兩個 `Task` 通過 `chain` 連線形成了一個非同步操作的序列——它先獲取 `user`，然後用 `user.id` 查詢 `user` 的 `friends`。`chain` 避免了 `Task(Task([Friend]))` 這種情況。

第二個例子是用 `querySelector` 查詢幾個 input 然後建立一條歡迎資訊。注意看我們是如何在最內層的函式裡訪問 `uname` 和 `email` 的——這是函式式變數賦值的絕佳表現。因為 `IO` 大方地把它的值借給了我們，我們也要負起以同樣方式把值放回去的責任——不能辜負它的信任（還有整個程式的信任）。`IO.of` 非常適合做這件事，同時它也解釋了為何 pointed 這一特性是 monad 介面得以存在的重要前提。不過，`map` 也能返回正確的型別：

```js
querySelector("input.username").chain(function(uname) {
  return querySelector("input.email").map(function(email) {
    return "Welcome " + uname.value + " prepare for spam at " + email.value;
  });
});
// IO("Welcome Olivia prepare for spam at olivia@tremorcontrol.net");
```

最後兩個例子用了 `Maybe`。因為 `chain` 其實是在底層呼叫了 `map`，所以如果遇到 `null`，程式碼就會立刻停止執行。

如果覺得這些例子不太容易理解，你也不必擔心。多跑跑程式碼，多琢磨琢磨，把程式碼拆開來研究研究，再把它們拼起來看看。總之記住，返回的如果是“普通”值就用 `map`，如果是 `functor` 就用 `chain`。

這裡我得提醒一下，上述方式對兩個不同型別的巢狀容器是不適用的。functor 組合，以及後面會講到的 monad transformer 可以幫助我們應對這種情況。

## 炫耀

這種容器程式設計風格有時也能造成困惑，我們不得不努力理解一個值到底嵌套了幾層容器，或者需要用 `map` 還是 `chain`（很快我們就會認識更多的容器型別）。使用一些技巧，比如重寫 `inspect` 方法之類，能夠大幅提高 debug 的效率。後面我們也會學習如何建立一個“棧”，使之能夠處理任何丟給它的作用（effects）。不過，有時候也需要權衡一下是否值得這樣做。

我很樂意揮起 monad 之劍，向你展示這種程式設計風格的力量。就以讀一個檔案，然後就把它直接上傳為例吧：

```js
// readFile :: Filename -> Either String (Future Error String)
// httpPost :: String -> Future Error JSON

//  upload :: String -> Either String (Future Error JSON)
var upload = compose(map(chain(httpPost('/uploads'))), readFile);
```

這裡，程式碼不止一次在不同的分支執行。從型別簽名可以看出，我們預防了三個錯誤——`readFile` 使用 `Either` 來驗證輸入（或許還有確保檔名存在）；`readFile` 在讀取檔案的時候可能會出錯，錯誤通過 `readFile` 的 `Future` 表示；檔案上傳可能會因為各種各樣的原因出錯，錯誤通過 `httpPost` 的 `Future` 表示。我們就這麼隨意地使用 `chain` 實現了兩個巢狀的、有序的非同步執行動作。

所有這些操作都是在一個從左到右的線性流中完成的，是完完全全純的、宣告式的程式碼，是可以等式推導（equational reasoning）並擁有可靠特性（reliable properties）的程式碼。我們沒有被迫使用不必要甚至令人困惑的變數名，我們的 `upload` 函式符合通用介面而不是特定的一次性介面。這些都是在一行程式碼中完成的啊！

讓我們來跟標準的命令式的實現對比一下：

```js
//  upload :: String -> (String -> a) -> Void
var upload = function(filename, callback) {
  if(!filename) {
    throw "You need a filename!";
  } else {
    readFile(filename, function(err, contents) {
      if(err) throw err;
      httpPost(contents, function(err, json) {
        if(err) throw err;
        callback(json);
      });
    });
  }
}
```

看看，這簡直就是魔鬼的算術（譯者注：此處原文是“the devil's arithmetic”，為美國 1988 年出版的歷史小說，講述一個猶太小女孩穿越到 1942 年的集中營的故事。此書亦有同名改編電影，中譯名《穿梭集中營》），我們就像一顆彈珠一樣在變幻莫測的迷宮中穿梭。無法想象如果這是一個典型的應用，而且一直在改變變數會怎樣——我們肯定會像陷入瀝青坑那樣無所適從。

# 理論

我們要看的第一條定律是結合律，但可能不是你熟悉的那個結合律。

```js
  // 結合律
  compose(join, map(join)) == compose(join, join)
```

這些定律表明了 monad 的巢狀本質，所以結合律關心的是如何讓內層或外層的容器型別 `join`，然後取得同樣的結果。用一張圖來表示可能效果會更好：

<img src="images/monad_associativity.png" alt="monad associativity law" />

從左上角往下，先用 `join` 合併 `M(M(M a))` 最外層的兩個 `M`，然後往左，再呼叫一次 `join`，就得到了我們想要的 `M a`。或者，從左上角往右，先開啟最外層的 `M`，用 `map(join)` 合併內層的兩個 `M`，然後再向下呼叫一次 `join`，也能得到 `M a`。不管是先合併內層還是先合併外層的 `M`，最後都會得到相同的 `M a`，所以這就是結合律。值得注意的一點是 `map(join) != join`。兩種方式的中間步驟可能會有不同的值，但最後一個 `join` 呼叫後最終結果是一樣的。

第二個定律與結合律類似：

```js
  // 同一律 (M a)
  compose(join, of) == compose(join, map(of)) == id
```

這表明，對任意的 monad `M`，`of` 和 `join` 相當於 `id`。也可以使用 `map(of)` 由內而外實現相同效果。我們把這個定律叫做“三角同一律”（triangle identity），因為把它圖形化之後就像一個三角形：

<img src="images/triangle_identity.png" alt="monad identity law" />

如果從左上角開始往右，可以看到 `of` 的確把 `M a` 丟到另一個 `M` 容器裡去了。然後再往下 `join`，就得到了 `M a`，跟一開始就呼叫 `id` 的結果一樣。從右上角往左，可以看到如果我們通過 `map` 進到了 `M` 裡面，然後對普通值 `a` 呼叫 `of`，最後得到的還是 `M (M a)`；再呼叫一次 `join` 將會把我們帶回原點，即 `M a`。

我要說明一點，儘管這裡我寫的是 `of`，實際上對任意的 monad 而言，都必須要使用明確的 `M.of`。

我已經見過這些定律了，同一律和結合律，以前就在哪兒見過...等一下，讓我想想...是的！它們是範疇遵循的定律！不過這意味著我們需要一個組合函式來給出一個完整定義。見證吧：


```js
  var mcompose = function(f, g) {
    return compose(chain(f), chain(g));
  }

  // 左同一律
  mcompose(M, f) == f

  // 右同一律
  mcompose(f, M) == f

  // 結合律
  mcompose(mcompose(f, g), h) == mcompose(f, mcompose(g, h))
```

畢竟它們是範疇學裡的定律。monad 來自於一個叫 “Kleisli 範疇”的範疇，這個範疇裡邊所有的物件都是 monad，所有的態射都是聯結函式（chained funtions）。我不是要在沒有提供太多解釋的情況下，拿範疇學裡各式各樣的概念來取笑你。我的目的是涉及足夠多的表面知識，向你說明這中間的相關性，讓你在關注日常實用特性之餘，激發起對這些定律的興趣。


## 總結

monad 讓我們深入到巢狀的運算當中，使我們能夠在完全避免回撥金字塔（pyramid of doom）情況下，為變數賦值，執行有序的作用，執行非同步任務等等。當一個值被困在幾層相同型別的容器中時，monad 能夠拯救它。藉助 “pointed” 這個可靠的幫手，monad 能夠借給我們從盒子中取出的值，而且知道我們會在結束使用後還給它。

是的，monad 非常強大，但我們還需要一些額外的容器函式。比如，假設我們想同時執行一個列表裡的 api 呼叫，然後再蒐集返回的結果，怎麼辦？是可以使用 monad 實現這個任務，但必須要等每一個 api 完成後才能呼叫下一個。合併多個合法性驗證呢？我們想要的肯定是持續驗證以蒐集錯誤列表，但是 monad 會在第一個 `Left` 登場的時候停掉整個演出。

下一章，我們將看到 applicative functor 如何融入這個容器世界，以及為何在很多情況下它比 monad 更好用。

[第 10 章: Applicative Functor](ch10.md)


## 練習

```js
// 練習 1
// ==========
// 給定一個 user，使用 safeProp 和 map/join 或 chain 安全地獲取 sreet 的 name

var safeProp = _.curry(function (x, o) { return Maybe.of(o[x]); });
var user = {
  id: 2,
  name: "albert",
  address: {
    street: {
      number: 22,
      name: 'Walnut St'
    }
  }
};

var ex1 = undefined;


// 練習 2
// ==========
// 使用 getFile 獲取檔名並刪除目錄，所以返回值僅僅是檔案，然後以純的方式列印檔案

var getFile = function() {
  return new IO(function(){ return __filename; });
}

var pureLog = function(x) {
  return new IO(function(){
    console.log(x);
    return 'logged ' + x;
  });
}

var ex2 = undefined;



// 練習 3
// ==========
// 使用 getPost() 然後以 post 的 id 呼叫 getComments()
var getPost = function(i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res({ id: i, title: 'Love them tasks' });
    }, 300);
  });
}

var getComments = function(i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res([
        {post_id: i, body: "This book should be illegal"},
        {post_id: i, body: "Monads are like smelly shallots"}
      ]);
    }, 300);
  });
}


var ex3 = undefined;


// 練習 4
// ==========
// 用 validateEmail、addToMailingList 和 emailBlast 實現 ex4 的型別簽名

//  addToMailingList :: Email -> IO([Email])
var addToMailingList = (function(list){
  return function(email) {
    return new IO(function(){
      list.push(email);
      return list;
    });
  }
})([]);

function emailBlast(list) {
  return new IO(function(){
    return 'emailed: ' + list.join(',');
  });
}

var validateEmail = function(x){
  return x.match(/\S+@\S+\.\S+/) ? (new Right(x)) : (new Left('invalid email'));
}

//  ex4 :: Email -> Either String (IO String)
var ex4 = undefined;
```
