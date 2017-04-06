# 第 1 章：我們在做什麼？

## 介紹

你好，我是 Franklin Risby 教授，很高興認識你。接下來我們將共度一段時光了，因為我要教你一些函數語言程式設計的知識。好了，關於我就介紹到這裡，你怎麼樣？我希望你已經熟悉 JavaScript 語言了，關於物件導向也有一點點的經驗了，而且自認為是一個合格的程式設計師。希望你沒有昆蟲學博士學位也能找到並殺死一些臭蟲（bug）。

我並不假設你之前有任何函數語言程式設計相關的知識——我們都知道假設的後果是什麼（譯者注：此處原文是“we both know what happens when you assume”，源自一句名言“When you assume you make an ASS of U and ME”，意思是“讓兩人都難堪”）。但我猜想你在使用可變狀態（mutable state）、無限制副作用（unrestricted side effects）和無原則設計（unprincipled design）的過程中已經遇到過一些麻煩。好了，介紹到此為止，我們進入正題。

本章的目的是讓你對函數語言程式設計的目的有一個初步認識，對一個程式之所以是*函數式*程式的原因有一定了解，要不然就會像無頭蒼蠅一樣，不問青紅皁白地避免使用物件——這等於是在做無用功。寫程式碼需要遵循一定的原則，就像水流湍急的時候你需要天文羅盤來指引一樣。

現在已經有一些通用的程式設計原則了，各種縮寫詞帶領我們在程式設計的黑暗隧道里前行：DRY（不要重複自己，don't repeat yourself），高內聚低耦合（loose coupling high cohesion），YAGNI （你不會用到它的，ya ain't gonna need it），最小意外原則（Principle of least surprise），單一責任（single responsibility）等等。

我當然不會囉裡八嗦地把這些年我聽到的原則都列舉出來，你知道重點就行。重點是這些原則同樣適用於函數語言程式設計，只不過它們與本書的主題不十分相關。在我們深入主題之前，我想先通過本章給你這樣一種感覺，即你在敲鍵盤的時候內心就能強烈感受到的那種函數式的氛圍。

<!--BREAK-->

## 一個簡單例子

我們從一個愚蠢的例子開始。下面是一個海鷗程式，鳥群合併則變成了一個更大的鳥群，繁殖則增加了鳥群的數量，增加的數量就是它們繁殖出來的海鷗的數量。注意這個程式並不是物件導向的良好實踐，它只是強調當前這種變數賦值方式的一些弊端。

```js
var Flock = function(n) {
  this.seagulls = n;
};

Flock.prototype.conjoin = function(other) {
  this.seagulls += other.seagulls;
  return this;
};

Flock.prototype.breed = function(other) {
  this.seagulls = this.seagulls * other.seagulls;
  return this;
};

var flock_a = new Flock(4);
var flock_b = new Flock(2);
var flock_c = new Flock(0);

var result = flock_a.conjoin(flock_c).breed(flock_b).conjoin(flock_a.breed(flock_b)).seagulls;
//=> 32
```

我相信沒人會寫這樣糟糕透頂的程式。程式碼的內部可變狀態非常難以追蹤，而且，最終的答案還是錯的！正確答案是 `16`，但是因為 `flock_a` 在運算過程中永久地改變了，所以得出了錯誤的結果。這是 IT 部門混亂的表現，非常粗暴的計算方式。

如果你看不懂這個程式，沒關係，我也看不懂。重點是狀態和可變值非常難以追蹤，即便是在這麼小的一個程式中也不例外。

我們試試另一種更函數式的寫法：

```js
var conjoin = function(flock_x, flock_y) { return flock_x + flock_y };
var breed = function(flock_x, flock_y) { return flock_x * flock_y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = conjoin(breed(flock_b, conjoin(flock_a, flock_c)), breed(flock_a, flock_b));
//=>16
```

很好，這次我們得到了正確的答案，而且少寫了很多程式碼。不過函式巢狀有點讓人費解...（我們會在第 5 章解決這個問題）。這種寫法也更優雅，不過程式碼肯定是越直白越好，所以如果我們再深入挖掘，看看這段程式碼究竟做了什麼事，我們會發現，它不過是在進行簡單的加（`conjoin`） 和乘（`breed`）運算而已。

程式碼中的兩個函式除了函式名有些特殊，其他沒有任何難以理解的地方。我們把它們重新命名一下，看看它們的真面目。

```js
var add = function(x, y) { return x + y };
var multiply = function(x, y) { return x * y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));
//=>16
```

這麼一來，你會發現我們不過是在運用古人早已獲得的知識：

```js
// 結合律（assosiative）
add(add(x, y), z) == add(x, add(y, z));

// 交換律（commutative）
add(x, y) == add(y, x);

// 同一律（identity）
add(x, 0) == x;

// 分配律（distributive）
multiply(x, add(y,z)) == add(multiply(x, y), multiply(x, z));
```

是的，這些經典的數學定律遲早會派上用場。不過如果你一時想不起來也沒關係，多數人已經很久沒複習過這些數學知識了。我們來看看能否運用這些定律簡化這個海鷗小程式。

```js
// 原有程式碼
add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));

// 應用同一律，去掉多餘的加法操作（add(flock_a, flock_c) == flock_a）
add(multiply(flock_b, flock_a), multiply(flock_a, flock_b));

// 再應用分配律
multiply(flock_b, add(flock_a, flock_a));
```

漂亮！除了呼叫的函式，一點多餘的程式碼都不需要寫。當然這裡我們定義 `add` 和 `multiply` 是為了程式碼完整性，實際上並不必要——在呼叫之前它們肯定已經在某個類庫裡定義好了。

你可能在想“你也太偷換概念了吧，居然舉一個這麼數學的例子”，或者“真實世界的應用程式比這複雜太多，不能這麼簡單地推理”。我之所以選擇這樣一個例子，是因為大多數人都知道加法和乘法，所以很容易就能理解數學可以如何為我們所用。

不要感到絕望，本書後面還會穿插一些範疇學（category theory）、集合論（set theory）以及 lambda 運算的知識，教你寫更加複雜的程式碼，而且一點也不輸本章這個海鷗程式的簡潔性和準確性。你也不需要成為一個數學家，本書要教給你的程式設計典範實踐起來就像是使用一個普通的框架或者 api 一樣。

你也許會驚訝，我們可以像上例那樣遵循函數式的典範去書寫完整的、日常的應用程式，有著優異效能的程式，簡潔且易推理的程式，以及不用每次都重新造輪子的程式。如果你是罪犯，那違法對你來說是好事；但在本書中，我們希望能夠承認並遵守數學之法。

我們希望去踐行每一部分都能完美接合的理論，希望能以一種通用的、可組合的元件來表示我們的特定問題，然後利用這些元件的特性來解決這些問題。相比命令式（稍後本書將會介紹命令式的精確定義，暫時我們還是先把重點放在函數式上）程式設計的那種“某某去做某事”的方式，函數語言程式設計將會有更多的約束，不過你會震驚於這種強約束、數學性的“框架”所帶來的回報。

我們已經看到函數式的點點星光了，但在真正開始我們的旅程之前，我們要先掌握一些具體的概念。

[第 2 章：一等公民的函式](ch2.md)
