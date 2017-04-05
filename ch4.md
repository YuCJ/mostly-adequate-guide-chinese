# 第 4 章: 柯里化（curry）

## 不可或缺的 curry

（譯者注：原標題是“Can't live if livin' is without you”，為英國樂隊 Badfinger 歌曲 *Without You* 中歌詞。）

我父親以前跟我說過，有些事物在你得到之前是無足輕重的，得到之後就不可或缺了。微波爐是這樣，智慧手機是這樣，網際網路也是這樣——老人們在沒有網際網路的時候過得也很充實。對我來說，函式的柯里化（curry）也是這樣。

curry 的概念很簡單：只傳遞給函式一部分引數來呼叫它，讓它返回一個函式去處理剩下的引數。

你可以一次性地呼叫 curry 函式，也可以每次只傳一個引數分多次呼叫。

```js
var add = function(x) {
  return function(y) {
    return x + y;
  };
};

var increment = add(1);
var addTen = add(10);

increment(2);
// 3

addTen(2);
// 12
```

這裡我們定義了一個 `add` 函式，它接受一個引數並返回一個新的函式。呼叫 `add` 之後，返回的函式就通過閉包的方式記住了 `add` 的第一個引數。一次性地呼叫它實在是有點繁瑣，好在我們可以使用一個特殊的 `curry` 幫助函式（helper function）使這類函式的定義和呼叫更加容易。

我們來建立一些 curry 函式享受下（譯者注：此處原文是“for our enjoyment”，語出自聖經）。

```js
var curry = require('lodash').curry;

var match = curry(function(what, str) {
  return str.match(what);
});

var replace = curry(function(what, replacement, str) {
  return str.replace(what, replacement);
});

var filter = curry(function(f, ary) {
  return ary.filter(f);
});

var map = curry(function(f, ary) {
  return ary.map(f);
});
```

我在上面的程式碼中遵循的是一種簡單，同時也非常重要的模式。即策略性地把要操作的資料（String， Array）放到最後一個引數裡。到使用它們的時候你就明白這樣做的原因是什麼了。

```js
match(/\s+/g, "hello world");
// [ ' ' ]

match(/\s+/g)("hello world");
// [ ' ' ]

var hasSpaces = match(/\s+/g);
// function(x) { return x.match(/\s+/g) }

hasSpaces("hello world");
// [ ' ' ]

hasSpaces("spaceless");
// null

filter(hasSpaces, ["tori_spelling", "tori amos"]);
// ["tori amos"]

var findSpaces = filter(hasSpaces);
// function(xs) { return xs.filter(function(x) { return x.match(/\s+/g) }) }

findSpaces(["tori_spelling", "tori amos"]);
// ["tori amos"]

var noVowels = replace(/[aeiou]/ig);
// function(replacement, x) { return x.replace(/[aeiou]/ig, replacement) }

var censored = noVowels("*");
// function(x) { return x.replace(/[aeiou]/ig, "*") }

censored("Chocolate Rain");
// 'Ch*c*l*t* R**n'
```

這裡表明的是一種“預載入”函式的能力，通過傳遞一到兩個引數呼叫函式，就能得到一個記住了這些引數的新函式。

我鼓勵你使用 `npm install lodash` 安裝 `lodash`，複製上面的程式碼放到 REPL 裡跑一跑。當然你也可以在能夠使用 `lodash` 或 `ramda` 的網頁中執行它們。

## 不僅僅是雙關語／咖哩

curry 的用處非常廣泛，就像在 `hasSpaces`、`findSpaces` 和 `censored` 看到的那樣，只需傳給函式一些引數，就能得到一個新函式。

用 `map` 簡單地把引數是單個元素的函式包裹一下，就能把它轉換成引數為陣列的函式。

```js
var getChildren = function(x) {
  return x.childNodes;
};

var allTheChildren = map(getChildren);
```

只傳給函式一部分引數通常也叫做*區域性呼叫*（partial application），能夠大量減少樣板檔案程式碼（boilerplate code）。考慮上面的 `allTheChildren` 函式，如果用 lodash 的普通 `map` 來寫會是什麼樣的（注意引數的順序也變了）：

```js
var allTheChildren = function(elements) {
  return _.map(elements, getChildren);
};
```

通常我們不定義直接運算元組的函式，因為只需內聯呼叫 `map(getChildren)` 就能達到目的。這一點同樣適用於 `sort`、`filter` 以及其他的高階函式（higher order function）（高階函式：引數或返回值為函式的函式）。

當我們談論*純函式*的時候，我們說它們接受一個輸入返回一個輸出。curry 函式所做的正是這樣：每傳遞一個引數呼叫函式，就返回一個新函式處理剩餘的引數。這就是一個輸入對應一個輸出啊。

哪怕輸出是另一個函式，它也是純函式。當然 curry 函式也允許一次傳遞多個引數，但這只是出於減少 `()` 的方便。

## 總結

curry 函式用起來非常得心應手，每天使用它對我來說簡直就是一種享受。它堪稱手頭必備工具，能夠讓函數語言程式設計不那麼繁瑣和沉悶。

通過簡單地傳遞幾個引數，就能動態建立實用的新函式；而且還能帶來一個額外好處，那就是保留了數學的函式定義，儘管引數不止一個。
下一章我們將學習另一個重要的工具：`組合`（compose）。

[第 5 章: 程式碼組合（compose）](ch5.md)

## 練習

開始練習之前先說明一下，我們將預設使用 [ramda](http://ramdajs.com) 這個庫來把函式轉為 curry 函式。或者你也可以選擇由 losash 的作者編寫和維護的 [lodash-fp](https://github.com/lodash/lodash-fp)。這兩個庫都很好用，選擇哪一個就看你自己的喜好了。

你還可以對自己的練習程式碼做[單元測試](https://github.com/llh911001/mostly-adequate-guide-chinese/tree/master/code/part1_exercises)，或者把程式碼拷貝到一個 REPL 裡執行看看。

這些練習的答案可以在[本書倉庫](https://github.com/llh911001/mostly-adequate-guide-chinese/tree/master/code/part1_exercises/answers)中找到。

```js
var _ = require('ramda');


// 練習 1
//==============
// 通過區域性呼叫（partial apply）移除所有引數

var words = function(str) {
  return split(' ', str);
};

// 練習 1a
//==============
// 使用 `map` 建立一個新的 `words` 函式，使之能夠操作字串陣列

var sentences = undefined;


// 練習 2
//==============
// 通過區域性呼叫（partial apply）移除所有引數

var filterQs = function(xs) {
  return filter(function(x){ return match(/q/i, x);  }, xs);
};


// 練習 3
//==============
// 使用幫助函式 `_keepHighest` 重構 `max` 使之成為 curry 函式

// 無須改動:
var _keepHighest = function(x,y){ return x >= y ? x : y; };

// 重構這段程式碼:
var max = function(xs) {
  return reduce(function(acc, x){
    return _keepHighest(acc, x);
  }, -Infinity, xs);
};


// 彩蛋 1:
// ============
// 包裹陣列的 `slice` 函式使之成為 curry 函式
// //[1,2,3].slice(0, 2)
var slice = undefined;


// 彩蛋 2:
// ============
// 藉助 `slice` 定義一個 `take` curry 函式，該函式呼叫後可以取出字串的前 n 個字元。
var take = undefined;
```
