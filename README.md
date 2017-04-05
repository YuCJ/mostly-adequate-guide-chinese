> 原作： *[mostly-adequate-guide](https://github.com/DrBoolean/mostly-adequate-guide)*, thank Professor [Franklin Risby](https://github.com/DrBoolean) for his great work!

> llh911001 簡中翻譯版：[JS函数式编程指南](https://www.gitbook.com/book/llh911001/mostly-adequate-guide-chinese/details)

> 繁體中文版由前簡中翻譯版透過 [OpenCC](https://github.com/BYVoid/OpenCC) 轉檔而來

<!-- <img src="images/cover.png"/> -->

# 關於本書

這本書的主題是函式正規化（functional paradigm），我們將使用 JavaScript 這個世界上最流行的函數語言程式設計語言來講述這一主題。有人可能會覺得選擇 JavaScript 並不明智，因為當前的主流觀點認為它是一門命令式（imperative）的語言，並不適合用來講函式式。但我認為，這是學習函數語言程式設計的最好方式，因為：

 * **你很有可能在日常工作中使用它**

    這讓你有機會在實際的程式設計過程中學以致用，而不是在空閒時間用一門深奧的函數語言程式設計語言做一些玩具性質的專案。

 * **你不必從頭學起就能開始編寫程式**

    在純函數語言程式設計語言中，你必須使用 monad 才能列印變數或者讀取 DOM 節點。JavaScript 則簡單得多，可以作弊走捷徑，因為畢竟我們的目的是學寫純函式式程式碼。JavaScript 也更容易入門，因為它是一門混合正規化的語言，你隨時可以在感覺吃力的時候回退到原有的程式設計習慣上去。

 * **這門語言完全有能力書寫高階的函式式程式碼**

    只需藉助一到兩個微型類庫，JavaScript 就能模擬 Scala 或 Haskell 這類語言的全部特性。雖然面向物件程式設計（Object-oriented programing）主導著業界，但很明顯這種正規化在 JavaScript 裡非常笨拙，用起來就像在高速公路上露營或者穿著橡膠套鞋跳踢踏舞一樣。我們不得不到處使用 `bind` 以免 `this` 不知不覺地變了，語言裡沒有類可以用（目前還沒有），我們還發明瞭各種變通方法來應對忘記呼叫 `new` 關鍵字後的怪異行為，私有成員只能通過閉包（closure）才能實現，等等。對大多數人來說，函數語言程式設計看起來更加自然。

以上說明，強型別的函式式語言毫無疑問將會成為本書所示範式的最佳試驗場。JavaScript 是我們學習這種正規化的一種手段，將它應用於什麼地方則完全取決於你自己。幸運的是，所有的介面都是數學的，因而也是普適的。最終你會發現你習慣了 swiftz、scalaz、haskell 和 purescript，以及其他各種數學偏向的語言。

### Gitbook (更好的閱讀體驗)

* [線上閱讀](https://yucj.gitbooks.io/mostly-adequate-guide-traditional-chinese/content/)
* [下載EPUB](https://www.gitbook.com/download/epub/book/yucj/mostly-adequate-guide-traditional-chinese)
* [下載Mobi (Kindle)](https://www.gitbook.com/download/mobi/book/yucj/mostly-adequate-guide-traditional-chinese)


# 目錄

## 第 1 部分

* [第 1 章: 我們在做什麼？](ch1.md)
  * [介紹](ch1.md#介紹)
  * [一個簡單例子](ch1.md#一個簡單例子)
* [第 2 章: 一等公民的函式](ch2.md)
  * [快速概覽](ch2.md#快速概覽)
  * [為何鍾愛一等公民](ch2.md#為何鍾愛一等公民)
* [第 3 章: 純函式的好處](ch3.md)
  * [再次強調“純”](ch3.md#再次強調“純”)
  * [副作用可能包括...](ch3.md#副作用可能包括)
  * [八年級數學](ch3.md#八年級數學)
  * [追求“純”的理由](ch3.md#追求“純”的理由)
  * [總結](ch3.md#總結)
* [第 4 章: 柯里化（curry）](ch4.md)
  * [不可或缺的 curry](ch4.md#不可或缺的-curry)
  * [不僅僅是雙關語／咖哩](ch4.md#不僅僅是雙關語咖哩)
  * [總結](ch4.md#總結)
* [第 5 章: 程式碼組合（compose）](ch5.md)
  * [函式飼養](ch5.md#函式飼養)
  * [pointfree](ch5.md#pointfree)
  * [debug](ch5.md#debug)
  * [範疇學](ch5.md#範疇學)
  * [總結](ch5.md#總結)
* [第 6章: 示例應用](ch6.md)
  * [宣告式程式碼](ch6.md#宣告式程式碼)
  * [一個函式式的 flickr](ch6.md#一個函式式的-flickr)
  * [有原則的重構](ch6.md#有原則的重構)
  * [總結](ch6.md#總結)

## 第 2 部分

* [第 7 章: Hindley-Milner 型別簽名](ch7.md)
  * [初識型別](ch7.md#初識型別)
  * [神祕的傳奇故事](ch7.md#神祕的傳奇故事)
  * [縮小可能性範圍](ch7.md#縮小可能性範圍)
  * [自由定理](ch7.md#自由定理)
  * [總結](ch7.md#總結)
* [第 8 章: 特百惠](ch8.md)
  * [強大的容器](ch8.md#強大的容器)
  * [第一個 functor](ch8.md#第一個-functor)
  * [薛定諤的 Maybe](ch8.md#薛定諤的-maybe)
  * [“純”錯誤處理](ch8.md#“純”錯誤處理)
  * [王老先生有作用...](ch8.md#王老先生有作用)
  * [非同步任務](ch8.md#非同步任務)
  * [一點理論](ch8.md#一點理論)
  * [總結](ch8.md#總結)
* [第 9 章: Monad](ch9.md)
  * [pointed functor](ch9.md#pointed-functor)
  * [混合比喻](ch9.md#混合比喻)
  * [chain 函式](ch9.md#chain-函式)
  * [理論](ch9.md#理論)
  * [總結](ch9.md#總結)
* [第 10 章: Applicative Functor](ch10.md)
  * [應用 applicative functor](ch10.md#應用-applicative-functor)
  * [瓶中之船](ch10.md#瓶中之船)
  * [協調與激勵](ch10.md#協調與激勵)
  * [lift](ch10.md#lift)
  * [免費開瓶器](ch10.md#免費開瓶器)
  * [定律](ch10.md#定律)
  * [總結](ch10.md#總結)


# 未來計劃

* 第 1 部分是基礎知識。這是初版草稿，所以我會及時更正發現的的錯誤。歡迎提供幫助！
* 第 2 部分講述型別類（type class），比如 functor 和 monad，最後會講到到 traversable。我希望能塞進來一些 monad transformer 相關的知識，再寫一個純函式的應用。
* 第 3 部分將開始遊走於程式設計實踐與學院學究之間。我們將學習 comonad、f-algebra、free monad、yoneda 以及其他一些範疇學概念。

