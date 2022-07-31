<!-- .slide: class="center" data-transition="none"-->
# Make Perl very fast

CindyLinz

2022.7.31

<aside class=notes>
大家好，這次的主題是，把 Perl 變得非常快。

主角是我寫的一個 perl module，它的用途是在不用改寫原始 Perl 程式的情況下，自動作一個預處理，把它變成 potentially 執行起來快好幾倍的程式。

我會介紹原本的 Perl 還有加速空間的部分，以及這個 module 實作的原理。
</aside>

---
<!-- .slide: data-transition="none"-->

## CindyLinz

  + 慧邦科技 gamesofa (神來也) 技術總監 <img style='margin-left:1em;width:4em;vertical-align:bottom' src=gamesofa.png>
  + <!-- .element: class="fragment" --> Ha<span class=fragment style=color:red;position:absolute;background-color:var(--r-background-color);margin-top:4px>ck</span>rdcore Perl 工程師
  + 我也很喜歡 C 與 Haskell <!-- .element: class="fragment" -->

<aside class=notes>
我是 Cindy，是神來也的工程師。
@@
技術上我是 hardcore @@ 아니 是 hackcore Perl 工程師
-- 同時我也是 hardcore Perl 工程師，아니@@，是 hackcore 工程師。

我喜歡在 Perl 的原始碼裡面翻來翻去，看什麼地方可以戳一戳，讓 Perl 擁有原本沒有的能力，而且可以和 Perl 原本的文化融合在一起。

-- 我喜歡挖開 Perl 的原始碼，揉捏 Perl 的內部結構。

最近寫的 Perl module 都有這樣的成份，包含今天要講的主角。

-- 我鑽進 Perl 裡面，成為 Perl 的一部分，讓 Perl 成為我肢體的延伸。

@@
另外我也很喜歡 C 和 Haskell。
-- 另外我也很喜歡 C 和 Haskell，喜歡嘗試一些 hacky way 的用法，這是人生的樂趣所在。
</aside>

---
<!-- .slide: data-transition="none"-->

## 程式語言的<span style=position:absolute>執行速度</span><span class=fragment data-fragment-index=1 style=background-color:var(--r-background-color)>設計哲學</span>

  | | |
  | - | - |
  | 快速組 | C, C++, C#, Rust, Java, Haskell,<br> Fortran, Pascal, Go, OCaml, F#, Ada, (Javascript) <p class=fragment data-fragment-index=1>對機器友善<br>(迎合硬體特性 <del>a.k.a 反人類</del>) |
  | 龜速組 | Perl, Ruby, Python, PHP, Erlang,<br> Racket, Lisp, (Javascript) <p class=fragment data-fragment-index=1> 對人類友善<br>(好上手 or 開發省事) |

<aside class=notes>
要找出加速 Perl 的方法，我們可以先看快的語言與慢的語言之間的特性差異。

首先先把程式語言分成快速組和龜速組。

這邊我是參考 benchmarks game 網站的資料，最快的是 C，然後用 C 的執行時間兩倍作為分界。

同一組內的排序我是隨便排的，沒有照速度來排。

Javascript 這邊被我打了括號，原因我後面會再提，而它剛好就差不多是 C 的兩倍。
@@
當然它們的設計目的就是不一樣的，為了對機器或是對人類友善而犧牲另一面，分化出不同的作法。

那我們就可以來找找兩組之間所採用的具體的不同手段是什麼。
</aside>

---
<!-- .slide: data-transition="none"-->

## Compile 與否？

  + <!-- .element: class="fragment" --> 其實 Perl 有 compile，Python、PHP 也都有 compile
  + <!-- .element: class="fragment" --> 重點是 compile 的時候做了些什麼

<aside class=notes>
第一個容易想到的分歧點是 compile。畢竟跑程式以前先花了一大堆時間 compile 了，那執行的時候應該會快一點才有天理嘛。
@@
其實，Perl 也有 compile，Python、PHP 也都有，所以這就不是分歧點。
@@
但同樣是 compile，它們做的事情不一樣多，花的時間也不一樣。

龜速組的程式語言為了開發方便，寫一寫就可以跑跑看，所以 compile 的時間在設計上盡量壓縮得很短。而快速組在 compile 階段就允許多做很多事。
</aside>

---
<!-- .slide: data-transition="none"-->

## OP code VS Native code？

  + <!-- .element: class="fragment" -->Native code 是 CPU 可以處理的指令編碼；<br>
    OP code 是相似位階的指令，不特別針對 CPU，執行時需要可以理解 OP code 的程式來處理。
  + <!-- .element: class="fragment" -->快速組的 Java 與 Haskell 也是 compile 成 OP code<br>
    (Java 的 GCJ 版本不算的話，反正實作也不完整)<br>
    (Haskell 可以把 thunk 視為廣義的 OP code)
  + <!-- .element: class="fragment" -->有一個 Perl module: Faster <a href=https://metacpan.org/pod/Faster>https://metacpan.org/pod/Faster</a><br>
    用於把 OP code "inline"，效能平均提升 20%<br>
    這的確會影響一部分效能，雖然幅度不是很大<br>
  + <!-- .element: class="fragment" -->OP code VS native 可以分辨出超級快速組 (C、C++、Rust)

<aside class=notes>
會是 OP code 與 Native code 的差別嗎？
@@
Native code 是 CPU 可以處理的指令編碼，不同血統的 CPU 不一樣，通常會剛剛好貼近電路結構。

所以同一份程式如果沒作特殊處理，像是把不同機器適用的指令碼通通打包在一起的話，放在不同血統的機器上是不能執行的。

OP code 是相似位階的指令，不特別貼近 CPU，執行時需要可以理解 OP code 的程式來處理。

這個程式通常是用 native code 做的，也有可能再間接一點，不過最後總是要回到某一層用 native 實作的程式。

OP code 執行起來當然會比 native 慢，不過還是比原始程式碼快。

額外地，一份 OP code 可以拿到各種機器上執行，只要該機器有準備能解讀它的軟體就可以。

另外 OP code 既然和硬體脫鉤，所以格式比較自由彈性，也常常會額外加上額外的資訊，例如原始檔名行號之類的 debug 可以用。
@@
不過快速組的 Java 和 Haskell 也是 compile 成 OP code，所以這個不算是關鍵的分界。
@@
Perl 有個 module，它會把 OP code 作 inline，就是把真正執行 OP code 的程式碼依 OP code 的順序拼接起來，重新 compile 成 native code 程式。

效能提升大概 20%，算是不可忽略的影響，雖然單靠這一項影響沒有很大。
@@
雖然 OP code 與 native code 的分別，不能區別快速組與龜速組，不過可以從快速組中分出超級快速組。
</aside>

---
<!-- .slide: data-transition="none"-->

## Garbage collection (GC)？

  + 執行環境會追蹤程式 allocate 的記憶體的使用情形<br>自動回收 "不會再用到" 的記憶體
  + <!-- .element: class="fragment" -->快速組的 Java 與 Haskell 也有記憶體自動回收機制
  + <!-- .element: class="fragment" -->garbage collection 的有無<br>也可以分辨出超級快速組 (C、C++、Rust) <!-- .element: class="fragment" -->

<aside class=notes>
GC 是執行環境提供的機制，選定一些方法來追蹤應用程式向系統要的記憶體，判斷這些記憶體還會不會再使用，在恰當的時機把不會再用的記憶體回收回來，以便之後再分派給應用程式使用。

這個追蹤的機制當然沒有100%準確，它判斷不會再用的記憶體會真的一定無法再使用，但是有些其實也不會再用的記憶體，它不一定看得出來。

GC 可以降低不少應用程式開發時的心理負擔，不過這個追蹤的機制是有代價的。
@@
類似地，快速組的 Java 與 Haskell 也有 GC，所以它也不是關鍵。
@@
而超級快速組就沒有 GC 了。
</aside>
---
<!-- .slide: data-transition="none"-->

## Undefined behaviors (UB)

<table width=80% class=fragment><tr>
<td width=33%><pre><code>if( x + 3 > 20 ){
  ...
}</code></pre>
<td width=33%><pre><code>if( x > 20 - 3 ){
  ...
}</code></pre>
<td><pre><code>if( x > 17 ){
  ...
}</code></pre>
<tr><td colspan=3 align=center><small>假設 x 是 32bits signed int</small>
</table>

  + <!-- .element: class="fragment" -->左邊的程式可以改成右邊的程式，節省一個加法嗎？
  + <!-- .element: class="fragment" -->當 x = 2147483647 (32bits signed int 的上限) 的時候呢？
  + <!-- .element: class="fragment" -->Undefined behaviors 解法的意思，就是程式語言文件裡註記，signed int 出現溢位的話，就不負責任何後果

<img class=fragment style=width:40% src=poor.jpg>

<aside class=notes>
Undefined behaviors 是一種陷阱，在程式語言的規格就寫，只要遇到某些情況，就不保證這個程式會幹出什麼事了。

C 和 C++ 的這種陷阱特別多，不過它竟然對效能提升是有益的。我們看看這個例子
@@
這三段程式看起來好像是一樣的東西。
@@
左邊的程式可以改成右邊嗎？ 省一個加法。

通常可以對不對，就是國中學到的移項變號。
@@
但如果，x 這時候的值是 2147483647，是 32bits signed int 的上限，再加3就破表了呢？

很多 native code 的有號整數，加到破表的時候溢位會從最小值繞回來，也就是一個負很大的值。

那麼它在左邊的程式不會大於20，但是在右邊的程式會大於17。如果 compiler 把左邊的程式 optimize 成右邊的程式，結果就錯了。
@@
UB解法就是：管你的我想改就改，你不准讓 x 吃到太大的數字就對了。

然後它們就在程式語言文件裡註記，signed int 出現溢位是一種 undefined behaviors。

這不是說溢位就會 crash，而是溢位就…… 規格上他也不知道會發生什麼事，compiler 也不會承諾說一定會把左邊改成右邊，只是有可能改，隨他高興。

實作 compiler 的人會假設這個情況一定不會發生來作設計。

而應用程式開發者，或是使用者要合力保證這樣的情境一定不會發生。

如果有錯，那就是你的錯。

C 和 C++ 就有極多的 undefined behaviors。

以前我還沒有小孩的時候，我把讀 C++ 的 undefined behaviors 當成消遣。

整數溢位算是比較單純的規則，程式碼上下看一看通常就可以判斷。

有的規則會牽涉多個角色，湊出爆炸的條件有一大半不在眼前，是不知道哪裡飛來的程式碼剛剛好作的配合，非常難搞。
@@
我一邊讀一邊笑，哇~ 寫 C++ 的人要整天搞這些東西哈哈哈看起來真可憐。

不過後來發現，寫 C++ 的人多半也沒全搞清楚就在寫了，然後就成為我們每天用的程式。

從開發者的角度來說，反正就算爆了，只要你的層級不高，不需要為整體成敗負政治責任。

這麼難避免的問題，主管要證明就是你造成的那也是不可能的任務。
</aside>

---
<!-- .slide: data-transition="none"-->

## Dynamic types？

  + 每一個變數都是萬用的，可以存放所有可能的資料
    ```perl
    if( $temperature =~ /^-?\d+\.?\d*℉$/ ) {
      $celcius = ($temperature - 32) * 5/9 . '℃';
    }
    if( $temperature =~ /^-?\d+\.?\d*℃$/ ) {
      $celcius = $temperature;
    }
    ```
  + <!-- .element class="fragment" -->每一次取值出來用的時候，要先判斷現在存的是哪一種資料
  + <!-- .element class="fragment" -->然後用對應的程式把值取出來<br>(可能會需要當場轉換格式，例如字串轉數字)
  + <!-- .element class="fragment" -->或是判斷要做哪一個版本的計算(Perl operator)<br>
    或是呼叫哪一個版本的函數(object oriented)

<aside class=notes>
Dynamic type 是人性化語言的重要特色，我們可以在還沒想清楚的情況下就開始試作程式。

每一個變數都是萬用的，可以存放所有可能的資料。

看這邊的例子，temperature 應該原本是放一個字串。

程式第一行比對它的內容是不是一串可能有負號的數字最後再加一個℉ ，是的話下面要把它轉成攝氏。

第二行要作轉換的時候，直接把它拿去作數學運算。Perl 文字轉數字的規則是從開頭略過空白以後找出數字樣的部分。

計算出來的數字再當成文字在結尾補上℃。

龜速組的程式語言大多有提供這樣的用法，這個人性化的設計的效能負擔是沈重的。
@@
顯而易見，每一次取值出來的時候，都要先判斷一下現在存的是哪一種資料。
@@
然後再使用對應的程式把資料取出來，而且可能要再執行轉換資料的程式，像上面把溫度字串轉存換為數字。
@@
如果要作計算的話，要找出這一次要使用哪一個版本的計算程式；

而如果是用 everything is an object 實作的語言，則是找出對應 class 的成員函數來處理。

對，每一次！每一次資料存取都要重複這些的動作。

不過同樣是 dynamic type，Perl 和其他語言有一點不太一樣：

其他語言的 dynamic type 裡，任何一個 value 都會是眾多 type 裡的其中一個，要嘛你是文字、要嘛你是數字、要嘛你是其他的某種東西。

而 Perl 在概念上沒有一個 value 歸屬哪個 type，只是取值使用的時候想把它當成什麼 type 來用。

如果有個 Perl value 是 37℃ ，如果拿來當文字用，就跟 3 7 ℃ 這幾個字一樣；如果拿來作數學計算，那它就跟三十七一樣。

當然 Perl 內部的實作還是有 type 的，可以處理文字和數字的 native 指令不一樣，接受的格式也不一樣，type 正確才能計算。

只是從使用者的角度，內部 type 怎麼處理正常是摸不到的，不只是寫 code 省事而已，它不會也無法成為心理的負擔。

我說正常摸不到 type，而不正常的情況就是我們現在要做的事，我們要討論 Perl 實作的效率影響，所以我們會把內部實作的 type 攤出來討論。

其他語言的 temperature 如果一開始存的是字串，在這段程式裡因為沒有寫入 temperature 的動作，所以 temperature 會一直維持是一個純粹的字串；

而 Perl 在把 temperature 拿來作數學計算時轉換出來的數字，會和原本的字串一起保存起來，爾後如果再要拿它作數字計算時就不需要重新轉換，直到 temperature 被寫入新值的時候，才會把它清除掉。在那之前轉換最多只會做一次。

反正 Perl 也沒有承諾過這個 temperature 是個文字，存下來的東西都是讓 Perl 在需要的時候可以變出文字或數字給你用的資訊。
</aside>

---
<!-- .slide: data-transition="none"-->

## Instruction pipeline

  + 五階段 instruction pipeline 範例<br>
    <img style=width:40% src=instruction-pipeline.png><br>
    <small>source: [wikipedia/指令管線化](https://zh.wikipedia.org/wiki/%E6%8C%87%E4%BB%A4%E7%AE%A1%E7%B7%9A%E5%8C%96)</small><br>
    <small>IF：讀取指令，ID：指令解碼，EX：執行，MEM：記憶體存取，WB：寫回暫存器</small>
  + <!-- .element: class="fragment" -->理想狀態下，拆五個階段可以五倍快，階段拆越多越快；<br>
    反過來說，程式碼不能連續執行的時候是理想狀態的五倍慢
  + <!-- .element: class="fragment" -->多作計算會慢一點，而多作「判斷」的影響會更嚴重
  + <!-- .element: class="fragment" -->硬體設計者發明一個加速的好設計，軟體設計者就會增加一堆麻煩要考慮……

<aside class=notes>
Dynamic type 要頻繁判斷 type 的影響不只是「多做一點計算」而已。

這個是 CPU 的 instruction pipeline 示意圖。每一個 native 指令在 CPU 執行的過程，可以拆成一些階段，不同的 CPU 可能會有不同的拆法，這一個例子是 wikipedia 上面給的五階段範例：

每一個指令都要經過從左到右 讀取指令、指令解碼、執行、記憶體存取、寫回暫存器 這五個階段。

在選定的拆分方式之下，不同的階段會巧妙地不佔用同一個 CPU 元件。

這樣就有榨取效能的空間了：在一個指令讀取完指令，要進入解碼階段的時候，下一個指令就可以先開始讀取指令了，反正讀取指令的元件閒著也是閒著
@@
你看，以這個拆成五階段的例子來說，執行效能就五倍快了。

常見的 PC CPU 也有拆七段、拆十幾段、拆三十一段的，而超級電腦有拆到一千段以上的。

段數越多加速倍數就越大，挑戰是要怎麼拆得出這麼多不會用到一樣元件的步驟。

BUT！效果要這麼好，是有條件的：CPU 要在前一個，不對，要在前四個指令還沒執行完以前，就能預判第五個指令是誰，才能順利偷跑。

那這四個指令裡，就不能有像是 if 意味的，不一定執行完以後下一個指令會跳到哪裡去的東西。

if 的情況稍微好一點，因為第三階段執行階段結束就知道結果了，所以只要知道前兩個指令沒有 if 就可以；

萬一出現了 if，還有一個作法是先假裝它不會發生跳躍，繼續偷跑接下來的指令，計算可以先作，結果先不要寫入就好，如果之後 if 真的決定不跳躍，那就算賺到。

但如果是函數呼叫，或是無條件跳躍一個動態的位址，那就一定不能執行接下來的指令，要先等位址算出來才能繼續。那就是紮實的五倍慢。
@@
總之，多做計算會多花時間，多做判斷或跳躍就會多花很多時間。
@@
硬體設計者每發明一個加速的好設計，軟體設計者就會增加一堆麻煩要考慮……

心中沒有加速的喜悅，只有沒做對就會降速的恐懼
</aside>

---
<!-- .slide: data-transition="none"-->

## Dynamic type variable 比較胖

  <img src=padded-var.png><br>
  <small>(此為示意圖，實際資料結構還有別的欄位)</small>

  + <!-- .element class="fragment" -->多佔用記憶體，對執行速度的影響比表面看起來的大

<aside class=notes>
再來就是 Dynamic type 的變數會比較胖。

左上是一個傳統的 static type 變數，除了資料本身以外，就沒有別的東西。

它下面兩個是存了不同 type 資料的 dynamic type 變數，除了資料以外，還要記錄現在的 type。

然後右邊是一個字串，由於字串的內容是可以增長的，所以內容部分需要一塊獨立的記憶體，然後再用指標連到它，這樣需要變長的時候，就可以直接再 allocate 一塊更大的記憶體，然後指標改指向新的這塊。

下面幾個是支援更多功能的字串。例如說如果常常要增長縮短，又不希望每一次都需要重新改變記憶體大小，那就要自己記錄一下真正用到的長度，後面多出來的記憶體當作備用區；

如果希望從前面移除字元的時候，不需要把整個字串往前搬，那就要記錄一下開頭的 offset，要空幾個字不用；

也可以挪用一部分的欄位，作為字串轉數字的暫存結果；

右下角這個是支援 copy-on-write 的情況。

這個 copy-on-write 是個很陰的手段，如果是寫低階網路服務，有很多字串轉移的動作，用 C++ 憑直覺正常寫而不動用正常人看不懂的手段，很難比 Perl 直覺寫的版本快。

啊~反正，支援的功能越多，就需要更多的欄位來記錄需要的資訊。
@@
那…多佔用記憶體，不僅僅是需要插更多記憶體才能跑的問題，它對執行速度的影響比表面看起來的還大。
</aside>

---
<!-- .slide: data-transition="none"-->

## Memory hierarchy

  <img src=padded-memory-hierarchy.png>

  + <!-- .element: class="fragment" -->往返 cache 與 memory 的次數，就幾乎是記憶體存取的總時間
  + <!-- .element: class="fragment" -->頻繁使用的「熱區」越小，被擠出 cache 的比例就越小
  + <!-- .element: class="fragment" -->memory cache 從 main memory 的存取<br>是以 32 或 64 bytes 為單位 (叫作 cache line)
  + <!-- .element: class="fragment" -->相鄰的 32 或 64 bytes 裡面，如果頻繁操作只用到其中 1 byte<br>效能就慢到最佳情況的 32 或 64 倍
  + <!-- .element: class="fragment" -->硬體設計者發明一個加速的好設計<br>軟體設計者就會增加一堆麻煩要考慮…… (again)

<aside class=notes>
這個是不同層級的記憶體媒體，最下面是主記憶體，越上是越靠近 CPU，速度越快容量也越小的儲存單元。

register 是 native 指令作計算時所使用的運算元，差不多可以視為 CPU 計算的速度。

這個資料是參考 2017 年的教科書。每一台機器的情況會不太一樣，不過大致相對的關係差不多都是這樣。

CPU 的研發有厲害的效能提升趨勢，但是記憶體的進步主要都在容量變大，效能的進展弱很多。
@@
從這個示意圖可以看到，資料往返 L1 cache 與主記憶體的次數，幾乎就是記憶體存取的總耗時，而 CPU 到 L1 cache 之間存取所花的時間，差不多是可以忽略的。
@@
所以，如果程式運行的過程中，使用的「熱區」越小的話，大部分的資料都可以塞在 cache 裡面不需要被擠出去，那效率就會大幅提升。

運氣足夠好的話，可以比頻繁進出 memory 與 cache 的程式快100倍。
@@
然後還不只是熱區總大小的問題。

這個 cache 的設計有個毛病，就是資料從主記憶體進出 cache 的時候，通常是必須以連續的 32 或 64 bytes 為單位，是 32 還是 64 視 CPU 型號而定。
@@
如果一次進來的 32 或 64 bytes 裡面，你只會用到其中的 1 byte，那你的效能就可能會比 利用連續 32 或 64 bytes 的演算法慢 32 或 64 倍。

因為人家每用 64 bytes 資料才需要搬一次，你每 1 byte 就要搬一次。

所以妥善安排資料排列的順序，讓排在一起的資料，使用時機剛好都在一起，那效率就會好得很多。沒排好效能就 GG。
@@
AGAIN，硬體設計者加了一個好設計，軟體設計者就會增加一大堆麻煩。

所以我從來不擔心什麼AI取代人類工作的問題，新發明在解決了現有的問題之餘，還會製造現在還不存在的問題。

而且製造的問題總是比解決的問題多，工業革命之後，以為機器可以代替人類工作，但實際上後來每個人的工作時間都變長了。
</aside>
---
<!-- .slide: data-transition="none"-->

## Boxed

  <img src=padded-boxed.png style=float:left>
  <img src=padded-var.png style=width:61%>

  + <!-- .element: class="fragment" -->紅色區域(global/stack)分配、回收很快，通常全部都在 cache 裡
  + <!-- .element: class="fragment" -->但是紅區初始化之後不能 resize、函數 return 以後就會被回收
  + <!-- .element: class="fragment" -->連續陣列裡的元素必須固定大小、同進同出

<aside class=notes>
boxed 是用比較鬆散的方式來配置資料的方法，右圖是前面舉例過的 Perl 內部資料結構示意圖。

其中字串為了保有伸縮的彈性，所以讓字串的內容部分使用獨立的記憶體。

boxed 可以視為這個概念的延伸。我們看左邊的示意圖

最上面是原始的，存著今天日期時間的陣列，每一組數字都緊緊靠在一起，我們只需要知道任意一個 address，再左右看看就可以存取每一組數字。
@@
並且這塊記憶體是放在程式可以直接拿的紅色區域，也就是整個程式全域變數區的特定位置、或是當前執行函數區域變數區的特定位置。

所以連 address 都不需要特別查詢，而是寫死在程式碼裡面，再加上全域、區域變數區開頭的 address 就可以拿到。

這是效率最好的配置方式，除了 address 可以立即取得之外，這兩個區域由於一定會頻繁使用的關係，通常都會位於 cache 裡面。

加上它們大小與相對位置固定，可以會在程式啟動、或函數進入的時候一步就把空間配置好，然後在函數結束時一步全回收，所以配置管理的成本也是非常低的。

但是它的彈性也是最差的：
@@
因為資料的前面跟後面都擠了別的資料，所以它不可以臨時擴大空間；

函數結束的時候因為會跟其他區域變數一起被回收，所以也不能把這塊資料留到函數結束之後使用。

而 boxed 就可以用來放寬這些限制。也就是左圖下面兩種作法，只在紅區保留一個存放 address 的空間，實際的資料放在跟系統要的動態記憶體。
@@
最下面這種作法，是在陣列元素的地方也再次使用這個技術。讓每一個陣列元素的大小都可以改變，而且可以擁有各自的生命週期，不需要在同時被回收。

看右邊這樣一堆例子，大小都是不一樣的，要讓變數可以動態改變它們的 type，就必須允許它們改變大小。

所以對 dynamic type 變數來說，boxed 幾乎是必用的。而且就用在最下面這種最徹底的用法，只有字串的內容部分，因為是umm 不需要單獨拆出其中一個字來轉 type。
那就可以用連續的記憶體
</aside>

---
<!-- .slide: data-transition="none"-->

## Boxed
  <img src=padded-boxed.png style=vertical-align:top;float:left>

  <table><tr><td>
  <ul>
    <li>存取時都要增加 dereference 的動作
    <li class=fragment>指標與記憶體管理所需要的資訊<br>可能比資料本體還胖
    <li class=fragment>獨立記憶體會分散在記憶體空間各處<br>降低 cache line 利用率
  </ul>
  </table>

<aside class=notes>
boxed 的成本是很高的，每一次存取都要額外作 dereference。
@@
然後這個 address 和記憶體管理所需要的資訊 可能比資料本體還要胖。

64bits 的機器上面，一個 address 就 8bytes 大，如果你只是要存 4bytes 的整數，就像是買了一個室內空間只有 1/3 的房子，另外 2/3 都填滿實心水泥。

而且這邊還沒有算上公設，就是記憶體管理的部分：系統要能回收記憶體，就需要有辦法知道這塊記憶體的大小，也需要記錄管理所有可以配發的記憶體，另外還有配發過程中所產生的像是畸零地一般怎樣都發不出去的記憶體浪費。
@@
另外，就是獨立分派的記憶體，會被記憶體管理服務任意分散在主記憶體空間各處而不會緊緊靠在一起，所以讀寫一組 dynamic type 變數內容的時候，cache line 的利用率也很低下。
</aside>

---
<!-- .slide: data-transition="none"-->

## Extreme OOP

  + 物件導向的 feature，把前述的效能禁手能犯的都犯了
    - <!-- .element: class="fragment" -->virtual function / method，只有在執行的時候才知道要呼叫誰<br>
      呼叫的時候判斷是什麼 class 的 instance 並呼叫對應的函數<br>(通常用 vtable 實作)
    - <!-- .element: class="fragment" -->整批的多型混合物件，需要 boxed pointer
    - <!-- .element: class="fragment" -->處理整批物件(就算是同型)，通常只會用到各物件裡同樣幾個少數欄位，所以 cache line 利用率低
      <br><img src=padded-Object.png>
  + <!-- .element: class="fragment" -->避開這些特性的用法叫作 POD(plain old data) type，如果程式語言可辨認出 POD 並改用特殊實作，可提升效能 (例如 C++)

<aside class=notes>
物件導向… 不知道為什麼，就剛剛好把前面提到的會降低效能的事情每一件都做足了。

一個比較有物件導向意味的程式，有好幾個 subclass 繼承共同的 superclass。

你可能有一個 superclass 的指標陣列，串起一堆 subclass 的物件，然後利用多型呼叫各物件有不同實作的同一個函數，讓各 subclass 實作的函數各自做各自該做的事。
@@
首先，這個 virtual function 每一次呼叫都要先判斷一下現在是什麼 class 該跳哪一個版本的實作，通常會是一個 vtable 的表格，裡面記載著要執行的函數的 address。

於是 instruction pipeline 的N倍加速就毀了。
@@
再來，各 subclass 可能會有自己的特色欄位，所以大小會不一樣，串成陣列通常都是使用 boxed 的方式。

於是記憶體的使用率也完了。
@@
接下來，由於物件通常是以個體的生機完整性組織起來的，而不是以任務導向來組織的。

舉個例子，這是一個RPG遊戲裡的怪物 class，每個物件裡會有一隻場上怪物的所有屬性，這隻怪物在遊戲中所有階段會用到的資訊都打包存在這裡：

有怪物頭上顯示的名稱、怪物的位置、大小，這是計算碰撞用的，有怪物的血量、MP、SP，身上的裝備，生成的時間，還有它中的異常狀態等等。

遊戲主程式的運行方式，會是週期性地在一個時機點，移動所有的怪物更新座標，也會用到 r 和 h 計算碰撞，這只會用到 x,y,z,r,h；過程中可能會觸發怪物攻擊，那會用到其他像是 hp,mp,sp，不過以每秒30~60個 frame 來看，每一個 frame 的週期裡絕大部分的怪物是不會出招的。

也就是說，在這一個 phase，我們會需要存取整排所有怪物的座標與大小欄位，然後很偶爾地會用到幾隻怪物的其他欄位。

物件的排列方式，一定是把一個物件的所有欄位都放在一起，然後再放下一個物件的所有欄位。

無可避免地，在一個 cache line 的 32 或 64bytes 裡面，除了我們要的 x,y,z,r,h 之外，還會塞一大堆這個 phase 用不到的資料。

這 phase 沒用到的欄位資料越大，資料存取的效能就越低落。

最佳安排應該是把所有的座標與大小資料拆出來，拼成一個只含所有怪物座標大小的陣列，然後其他欄位再自己放一個陣列。

當然，如果真的這樣安排，那物件就不物件了，物件導向 style 的封裝什麼的，都破滅了。會有很多原本的 member function，變成要同時讀取兩個遙遠物件的內容才能運作的函數，不知道它該算是誰的 member 才對。
@@
如果可以不用到物件導向裡的繼承、virtual function 這些東西，這種物件叫作 POD，是 plain old data 的縮寫。

如果程式語言可以辨認出 POD 形式的物件，並且搭配特殊的實作，就可以用效能比較好的方式來使用它。

像 C++ 就有這樣的設計。其實就是剛好 C struct 可以實作的收斂情況，只是使用物件的語法來寫出根本就不是物件的東西。
</aside>

---
<!-- .slide: data-transition="none"-->

## Extreme OOP

  + 物件導向基本上就是效能殺手；<br>"極端" 物件導向，就是把效能壓在地上摩擦

<table style=width:90% class=fragment>
  <tr><td>Java (物件導向)<td>Python (極端物件導向)
  <tr>
    <td>
<pre><code class=language-java>public int triangle(){
  int sum = 0;
  for(int i=1; i<=this.n; ++i)
    sum += i;
  return sum;
}
</code></pre>
    <td>
<pre><code class=language-python>def triangle(self):
  sum = 0
  for i in range(1, self.n):
    sum += i
  return sum
</code></pre>
</table>

<aside class=notes>
物件導向基本上就是效能殺手；"極端" 物件導向，就是把效能壓在地上摩擦
@@
我們比較一下 Java 與 Python，解釋為什麼 Java 雖然也是物件導向，但是仍然可以齊身快速組。

左右兩邊的程式都是叫作 triangle 的 member function，做的事情是一樣的，就是從 1 加到 member field n。

它們一樣是物件的成員函數，但是函數裡面的 sum 和 i：

在 Java 這邊是 int 整數，是 primitive type，沒有用到 boxed 技術，直接在函數的區域變數區裡放兩個整數；

而 Python 這邊的 i 和 sum 都是物件，那個 += 是在呼叫 virtual function。

所以，雖然同樣都叫作物件導向的程式語言。在 Java 呼叫一次 triangle 會觸動一次 virtual function dispatch，而在 Python 會觸動至少 n+1 次，還沒加計 range 取值n次的開銷。

當 Java 自砍一次效能，Python 就砍 n+1 次。這就是極端物件導向。
</aside>

---
<!-- .slide: data-transition="none"-->

## Extreme OOP

  + Perl 的禮物，就是它是在物件導向流行起來以前發展的
    | | released |
    | - | :-: |
    | Perl | 1988 |
    | Python | 1991 |
    | Perl5 | 1994 |
    | Java | 1995 |
    | Ruby | 1995 |
    | Python2 | 2000 |
    | Python3 | 2008 |
  + <!-- .element: class="fragment" -->雖然她沒有利用這個特點來提升效能
  + <!-- .element: class="fragment" -->所以我才有機會寫這個 module

<aside class=notes>
Perl 的禮物，就是它是在物件導向流行起來以前發展的。

第一版 Perl 是 1988 年，不過現在用的都是 Perl5，1994，還是比物件導向呱呱叫的 Java 早一年。

1991 年的 Python 也是沒有物件的，不過從 2000 年開始有物件的 Python2 才流行起來。

後繼的 Python、Ruby 都是在物件導向旋風颳起來以後發展的，它們一起頭就得意地說 everything is an object，這根本就是詛咒。
@@
雖然 Perl 沒有利用這個特點來提升效能，只有輔助到程式碼的可讀性。
@@
嗯，所以我才有機會寫這個 module 嘛。
</aside>

---
<!-- .slide: data-transition="none"-->

## 關鍵就是 static type

  + <!-- .element: class="fragment" --> 雖然快速組 C、C++、Java、Haskell 也都有 dynamic type
    - C 和 C++ 有 void\*
    - Java 有 class Object
    - Haskell 有 data Dynamic
  + <!-- .element: class="fragment" -->雖然只要程式語言有物件<br>所有的 parent class 都是廣義的 dynamic type
  + <!-- .element: class="fragment" -->但它們不是 "extreme" dynamic type<br>執行到相關指令的頻率低，效能影響佔比小

<aside class=notes>
影響非常大的關鍵就是 static type 與 dynamic type 的差別。
@@
雖然技術上來說，快速組的 C、C++、Java、Haskell 也都有 dynamic type。
@@
除了像 void星、Object class 這種很純的動態型別，其實物件導向裡所有的 parent class 也都是廣義的 dynamic type。
@@
但它們不是 EXTREME dynamic type，那執行到相關指令的頻率低，影響就小了。
</aside>
---
<!-- .slide: data-transition="none"-->

## 如何讓 Perl 有型(type)

<img src=lion-king1.png style=width:32%;vertical-align:middle>
<img class=fragment src=lion-king2.png style=width:64%;vertical-align:middle>

  + 希望 type 不用自己標 <!-- .element: class="fragment" -->
  + 不然寫 C 就好了 <!-- .element: class="fragment" -->

<aside class=notes>
既然問題是 type，那如何加上 type？ 要求開發者標 type 嗎？
@@
雖然想執行快，我還是想寫得很快。 標了 type 要怎麼快？
@@
所以我希望 type 還是不用自己標。
@@
不然我寫 C 就好了嘛！
</aside>

---
<!-- .slide: data-transition="none"-->

## Javascript 的作法：JIT compiler

  + <!-- .element: class="fragment" -->假設 dynamic type 的變數，實際上有很多不常一直變動 type
  + <!-- .element: class="fragment" -->先跑一陣子，統計一下它們實際上都是什麼 type
  + <!-- .element: class="fragment" -->然後再 compile 出特定 type 的版本
  + <!-- .element: class="fragment" -->執行的時候確認一下有沒有猜對
  + <!-- .element: class="fragment" -->這是我在程式語言分組時把 JS 打括號的原因

<aside class=notes>
參考一下 Javascript 的作法，或說是 node.js 的作法，是用 Just in time compile
@@
假設 dynamic type 的變數，實際上有很多不常變動 type，這應該是符合預期
@@
先跑一陣子，統計一下它們實際上都是什麼 type
@@
然後再 compile 出特定 type 的版本
@@
執行的時候就只確認一下有沒有猜對，確認一次可以用一整片程式，因為在一大段程式裡面有沒有 type 轉換，是從程式碼和進入點的 type 可以先判斷出來的。
@@
這是我在程式語言分組時把 JS 打括號的原因。因為它需要先跑慢速版本收集資訊，然後跑到一半再製造快速版本繼續跑
</aside>

---
<!-- .slide: data-transition="none"-->

## Perl 可以用的作法：type inference

  + <!-- .element: class="fragment" -->雖然 Perl 的 variable 是 dynamic type<br>但 operator 是 static type
  + <!-- .element: class="fragment" -->operator 會提示我們，一個變數
    - 只會被當成哪個(些) type 使用
    - 只會得到哪個(些) type 的值
    ```
    sub fib {
        my($n) = @_;
        my($a, $b) = (1, 1);
        while( --$n >= 0 ) {
          ($a, $b) = ($b, $a+$b);
        }
        return $a;
    }
    ```
  + <!-- .element: class="fragment" -->對照組 (Javascript)
    ```javascript
    console.log(a + 0)
    // 會印出 a 的原值，還是十倍大的 a？
    ```

<aside class=notes>
Perl 可以作 type inference。Perl 的特性還特別適合 type inference。
@@
雖然 Perl 的 variable 是 dynamic type，但 operator 是 static 的，還帶著 type 資訊。
@@
operator 會提示我們，一個變數只會被當成哪些 type 使用，只會得到哪些 type 的值。我們看這個例子：

這是計算費氏數的函數。

這個 $n 是傳進來的參數，它會是什麼 type 的內容我們不知道，但是看到下面 $n 只會拿來作 -- 的計算，並且計算結果會被拿來比大小。

注意 Perl 的 >= 一定是數值的比較，如果 $n 是原本字串，甚至就算右邊的 0 也是給字串型式，遇到 >= 就是把兩邊都當成數值來比較。

所以我們可以確認說，$n 只會被當成數值使用。

那麼，其實函數一開始執行的時候，就可以把這個不知道裡面裝了什麼的 $n 先轉換成數字，然後放在一個靜態的數值變數裡面，爾後所有的計算都只存取這個靜態的數值變數就好。

$a 和 $b 也可以作類似的推導，Perl 的 + 號一定都是數值加法，無論兩邊原本是數值還是文字，通通都會當成數值來加。

我們也可以得出結論，$a 和 $b 都只需要存數字。雖然最後面 return $a 之後，拿了它的值的人會當什麼用我們不知道，不過我們只要在最後一刻，再生出 dynamic type 的值就可以了。
@@
對照一下 Javascript，這邊光看 a + 0，我們就看不出來 console.log 會印出 a 還是十倍的 a，因為這個加號需要根據 a 的 type 來決定會作數值加法還是字串相接。
</aside>

---
<!-- .slide: data-transition="none"-->
## Perl 可以用的作法：type inference

  + 更多有提示性的 operator　　　　　　　　　
    ```
    my $content;
    for my $chapter (@chapter) {
      $content .= $book{$book_name}[$chapter];
    }
    my $char_count = length $content;
    return "there are $char_count characters";
    ```
  + <!-- .element: class="fragment" -->Perl 所有的 builtin function 通通都是 operator

<aside class=notes>
Perl 除了一般 binary operator 以外，還有更多的提示性的 operator。

這邊，$book{$book\_name}，表明 $book 一定是一個 hash (就類似 python 的 dict)，而 $book\_name 無論原本是什麼，在這裡一定會被當文字使用。

後面的 [$chapter]，表明前面的 $book{$book\_name} 會是一個 array 的 reference，然後 $chapter 會被當整數使用。

下面，$char\_count 等於 length $content，這表明 $content 會被當成文字使用，然後 $char\_count 會被存入非負整數。

Perl 裡所有內建函數其實都是 operator，都有專屬的編碼。
</aside>

---
<!-- .slide: data-transition="none"-->
## Perl 可以用的作法：type inference
  + static operator 是 Perl 一貫的精神。曾經走偏過，又修正回來：
    - each operator
      ```
      while(my($key, $value) = each %hash) { ... }
      ```
    - Perl 5.12: 新增 each operator on array
      ```perl
      while(my($index, $value) = each @array) { ... }
      ```
    - Perl 5.14: 新增 each operator on reference (experimental)
      ```
      while(my($index_or_key, $value) = each $ref_to_ary_or_hash) { ... }
      ```
    - Perl 5.24: 刪除 each operator on reference

    <small>(目前 Perl current 是 5.36)</small>

    這也是 sigil $@%&* 在 Perl 很重要的原因

<aside class=notes>
static operator 是 Perl 一貫的精神。雖然曾經走偏過，後來又再改正。

以這個 each 為例，它原本是放在 while loop 裡，用來 iterate 一個 hash 的 key 與 value。

到了 5.12 版，擴充它可以 iterate array 的 index 與 value。

這樣好像 each 變成了 dynamic operator，會根據參數的 type 不同而有不同的行為。

但是因為參數是 hash 還是 array，是從它開頭的符號 % 或 @ 可以判斷的，語法限制一定要寫出來。

如果把 each 與參數的開頭符號一起看，可以簡單視為兩個不同的 static operator，還不算破壞原則。

然而到 5.14 版的時候，它又進一步擴充到可以 iterate reference。

反正 array 和 hash 都能做了，多一步做它們的 reference。

欸，不過這一次就真的破壞原則了，從 reference 的開頭符號看不出來它背後的是 array 還是 hash，這就真的變成 dynamic operator 了。

於是到了 5.24 版就決定把 each on reference 的用法刪除，反正這段期間 each on reference 都是註明為 experimental，保持隨時會收回的彈性。

這邊也可以順便看到這個叫作 sigil 的開頭符號，在 Perl 是有重要意義的。

像 PHP 這個一出生就定位為簡化版 Perl 的語言，把 sigil 簡化到只剩 $號，就遺失了很多 Perl static operator 的特性。

Python、Ruby 設計時參考了許多 Perl 的特性，我覺得很可惜的就是沒有把 static operator 的特性學走。

它們都走上了極端物件導向的路，嘖嘖嘖嘖嘖嘖
</aside>

---
<!-- .slide: data-transition="none"-->

## Perl 的 OP tree

站在 Perl 的肩膀上，不用自己 parse 一遍

```perl
while( --$n >= 0 ) {
  ($a, $b) = ($b, $a+$b);
}
```
<img src=padded-optree.png>

<aside class=notes>
剛剛舉例推論變數 type 的時候，我們是用眼睛對著程式碼作分析。

不過我實作這個 module 的時候，不是直接對著程式碼做的。

前面說過 Perl 本身是會 compile 的，所以我可以利用 Perl 自帶的 compiler，不用自己再 parse 一遍。

老實說，Perl 有些語法 parse 起來真的很麻煩。

下圖是這段 code，就是計算費氏數的迴圈，經過 Perl compile 出來的 OP code 結構。

它是一個樹狀結構，其結構所含的資訊可以重新復原出原本的程式碼，不過原有的排版方式與註解是沒有的。
</aside>

---
<!-- .slide: data-transition="none"-->

## Perl 的 OP tree (linked)

不用自己 parse 一遍，而且還附贈 optimize

```
while( --$n >= 0 ) {
  ($a, $b) = ($b, $a+$b);
}
```
<img src=padded-optree2.png>

<aside class=notes>
這個 OP code 的樹狀結構還有附上執行順序，而且是 optimize 過的版本，有些節點是經判斷可以直接省掉不執行的。

我會拿這個 optimize 過的 OP tree 來使用。
</aside>

---
<!-- .slide: data-transition="none"-->

## Perl 的 OP code 與 stack

OP code 執行的時候，是操作存取 stack 上的資料

```
($a, $b) = ($b, $a+$b);
```
<img src=padded-op-stack.png>

<aside class=notes>
OP code 在執行的時候，除了會決定下一個 OP code 是誰以外，只會這樣子跟系統 stack 作互動，不會和其他的 OP code 來往。

這邊例子是剛剛的迴圈的 body 部分：

先在 stack 上面放一個 mark，準備後面要做的 list 對 list 的 assign。

等號右邊的 $b 會先放在 stack 上，然後是準備要加法計算的 $a 與 $b。

作了加法之後，就把右邊的 $a 和 $b 從 stack 移除，然後放入加完的結果。

下面把準備要接收新值的 mark、$a、$b 放上 stack。

然後用 mark 定位，作一對一的 copy。
</aside>

---
<!-- .slide: data-transition="none"-->

## 建構 type inference 條件式

<img src=padded-op-stack.png>
<img src=padded-op-add.png>

<aside class=notes>
這個計算的連結，我們可以作出右邊這個很像電子元件圖的東西。

這邊有三個元件，$a、$b 和加法。

電子元件我不知道加法習慣用什麼符號，所以我就用 XOR 代替。

這三個元件左邊是 input 右邊是 output，大部分都是依左圖的箭頭作連結，把一個 output 餵給另一個 input。

比較特別的只有 $b 的 output。因為它有兩個用途，一個是 assign 給 $a，另一個是作為加法的 input，所以元件圖裡 $b 的 output 出去分為兩個。
</aside>
---
<!-- .slide: data-transition="none"-->

## 建構 type inference 條件式

<img src=padded-op-add.png>

`\[
  \begin{aligned}
    a_{out} &= add_{in1}                  &\$a &= a_{in} \cap a_{out} \\
    b_{out} &= add_{in2} \cup a_{in}      &\$b &= b_{in} \cap b_{out} \\
    add_{out} &= b_{in} \\
  \end{aligned}
 \]
 \[
  \begin{aligned}
    add_{in1}, add_{in2}, add_{out} &\subset \{INT, NUM\} \\
    add_{out} &\subset \{INT\} \ if \  add_{in1} \cup add_{in2} \subset \{INT\}
  \end{aligned}
\]`

<aside class=notes>
從元件圖，我們可以再寫出下面的式子。

左上的式子反映的是元件圖裡面箭頭連線，岔路反映的就是聯集的關係，意味著就是都有可能發生。

右上式子反映的是…變數需要保存的資料，只有 input 與 output type 的交集部分。

超過 input 的部分反正拿不到、超過 output 的部分反正沒有人想知道。

下面的式子反映的是加法 operator 自己天生只想吃數值，也只會輸出數值的特性。這是第一式。

第二式是考慮到數值其實有分整數與浮點數，如果 input 都是整數的話，輸出就一定是整數。

第二式是不一定要做的，只是如果能做得細緻一點，有機會把各變數的 type 限縮得更小，那產生出來的程式碼就會更有效率。

那…有了式子就可以求解。

我求解的方式是先假設所有的 input、output type 都是 ANY。

然後逐個式子去看會不會縮小範圍，例如說加法的 input、output 遇到下面的式子就會從 ANY 縮小為整數或浮點數。

然後它們會再影響到 $a、$b 的 input、output。

由於 $a 和 $b 只會保存它們 input、output 的交集部分，所以無論式子裡需要引用它們的 input 或 output，我都直接引用 input 與 output 目前的交集部分這樣。

那就一直重複這過程到所有的 type 都不再變化為止。
</aside>
---
<!-- .slide: data-transition="none"-->

## 依判斷出來的 type 生成 C code

<pre><code>L16:; // nextstate
L17:; // pushmark
L18:; // padsv
L19:; // padsv
L20:; // padsv
L21:; // add
IVNV_t t10; // (INTNUM)
t10 = pad4;
IVNV_t t11; // (INTNUM)
t11 = pad5;
IVNV_t pad7; // (INTNUM)
int t10_is_int = t10.INT!=IV_MIN; // INTNUM
int t11_is_int = t11.INT!=IV_MIN; // INTNUM
if( t10_is_int && t11_is_int ){
  IV a_int, b_int, o_int;
if( t10.INT != IV_MIN )
  a_int = t10.INT;
else
  a_int = (IV)(t10.NUM + .5);
if( t11.INT != IV_MIN )
  b_int = t11.INT;
else
  b_int = (IV)(t11.NUM + .5);
  o_int = a_int + b_int;
pad7.INT = o_int;
}else{
  NV a_num, b_num, o_num;
if( t10.INT != IV_MIN )
  a_num = (NV)t10.INT;
else
  a_num = t10.NUM;
if( t11.INT != IV_MIN )
  b_num = (NV)t11.INT;
else
  b_num = t11.NUM;
  o_num = a_num + b_num;
pad7.INT = IV_MIN;
pad7.NUM = o_num;
}
L22:; // padrange
L23:; // aassign
IVNV_t t12; // (INTNUM)
t12 = pad5;
pad4 = t12;
IVNV_t t13; // (INTNUM)
t13 = pad7;
pad5 = t13;
</code></pre>

  + <!-- .element: class="fragment" --> 豪邁地使用很多區域變數來降低組織程式碼的複雜程度
    - OP code 元件的 in/out 接點各用一個變數
    - 接點往 OP code 元件內部的轉型也各用一個變數
    - 用作判斷 type flag 的變數，有時其實是常數，仍放到 if 判斷裡
  + <!-- .element: class="fragment" --> 反正 gcc 的 optimizer 很厲害，會把多餘的東西收整回來

<aside class=notes>
type 決定好以後的下一步，就是根據這些 type 把對應的 C code 產生出來。

這一串就是剛剛 list assign 與加法所對應的 C code。應該沒有人想要細看。
@@
生成的 code 裡面，我豪邁地使用了很多區域變數來降低組織程式碼的複雜度。

包括所有元件的 input、output 都用一個變數，轉型給元件內部使用也用一個變數，要判斷 type 的 flag 也用一個變數，即使它有時候根本就是常數。
@@
反正 gcc 的 optimizer 很厲害。
</aside>
---
<!-- .slide: data-transition="none"-->

## 用法與效果比較

<table width=100%><tr>
  <td>
    <pre><code>sub fib { ... }
...</code></pre>
  <td>
    <pre><code>use Inline;
Inline->bind(Speedup => 'fib');
sub fib { ... }
...</code></pre>
  <tr class=fragment>
    <td><pre>fib(1) = 1 in 0.0000 seconds
fib(2) = 2 in 0.0000 seconds
fib(3) = 3 in 0.0000 seconds
fib(4) = 5 in 0.0000 seconds
fib(5) = 8 in 0.0000 seconds
fib(85) = 420196140727489673 in 0.0000 seconds
fib(86) = 679891637638612258 in 0.0000 seconds
fib(87) = 1100087778366101931 in 0.0000 seconds
fib(88) = 1779979416004714189 in 0.0000 seconds
fib(89) = 2880067194370816120 in 0.0000 seconds
fib(90) = 4660046610375530309 in 0.0000 seconds</pre>
    <td><pre>fib(1) = 1 in 0.0000 seconds
fib(2) = 2 in 0.0000 seconds
fib(3) = 3 in 0.0000 seconds
fib(4) = 5 in 0.0000 seconds
fib(5) = 8 in 0.0000 seconds
fib(85) = 420196140727489673 in 0.0000 seconds
fib(86) = 679891637638612258 in 0.0000 seconds
fib(87) = 1100087778366101931 in 0.0000 seconds
fib(88) = 1779979416004714189 in 0.0000 seconds
fib(89) = 2880067194370816120 in 0.0000 seconds
fib(90) = 4660046610375530309 in 0.0000 seconds</pre>
</table>
<div class=fragment>N 再加大，64bits 整數就存不下結果了</div>

<aside class=notes>
這個 module 的用法就是使用 Ingy 寫的 Inline module 作為入口，然後指定要 Speedup 哪些函數。

也可以指定要傳給 gcc 的 compile 參數，不過我這邊就不寫出來佔版面了。

好我們來比較效果。
@@
好這個 case 太快了比不出效果。
@@
而且 N 不能再大了，再大就存不下了。
</aside>
---
<!-- .slide: data-transition="none"-->

## 用法與效果比較

<table><tr><td colspan=2>
<pre><code>sub count_factor {
    my($n) = @_;
    my $n = int $n;
    my $ans = 0;
    for(my $i=1; $i<=$n; $i+=1) {
        if( $n % $i == 0 ) {
            ++$ans;
        }
    }
    return $ans;
}</code></pre>
<tr class=fragment>
  <td><pre># of factors of 10 is 4 in 0.0000 seconds
# of factors of 20 is 6 in 0.0000 seconds
# of factors of 1000 is 16 in 0.0001 seconds
# of factors of 10000 is 25 in 0.0005 seconds
# of factors of 1048576 is 21 in 0.0588 seconds
# of factors of 16777216 is 25 in 0.8685 seconds
# of factors of 67108864 is 27 in 3.4866 seconds</pre>
  <td><pre># of factors of 10 is 4 in 0.0000 seconds
# of factors of 20 is 6 in 0.0000 seconds
# of factors of 1000 is 16 in 0.0000 seconds
# of factors of 10000 is 25 in 0.0001 seconds
# of factors of 1048576 is 21 in 0.0081 seconds
# of factors of 16777216 is 25 in 0.1247 seconds
# of factors of 67108864 is 27 in 0.4853 seconds</pre>
</table>
<div class=fragment>$\frac{3.4866}{0.4853} = 7.1844$ 倍</div>

<aside class=notes>
我們換一個 case，這個是計算輸入參數有幾個正因數。

計算的方式簡單粗暴，就是從 1 到 N 通通除除看。

函數第二行有個 int 是為了讓 type inference 能看出這個輸入參數只會被當整數使用，就算是浮點數也先轉整數再用。
@@
好，這個算得比較久就可以比結果。
@@
效能進展大概七倍多。
</aside>
---
<!-- .slide: data-transition="none"-->

## 目前狀態 - POC

  + github repo [https://github.com/CindyLinz/Perl-Speedup/](https://github.com/CindyLinz/Perl-Speedup/)

  + <!-- .element: class="fragment" --> Perl 的 OP code 很多<br>(Perl 5.30 有 398 個、Perl 5.36 有 415 個)<br>
    目前只有實作測試會用到的、並順手再多做一些些<br>
    還需要一個一個把其他 code gen 實作補齊

  + <!-- .element: class="fragment" -->程式寫得趕還沒整理乾淨，包括 1558 lines 長的 gen_code 函數。<br>
    別人想幫忙增改大概都不太可能…

<aside class=notes>
那…這個 module 完成度還不高，雖然已經可以執行了，而且有部分的成果。
@@
不過因為 Perl 的 OP code 實在很多，最新版有 415 個，舊一點 5.30 也有 398 個。

我現在只有實作測試有用到的 OP code，然後順手再多做幾個。

還有很多 OP code 的 code gen 需要補。
@@
另外就是為了想要趕 COSCUP 的日程，所以寫得很趕，程式碼都是隨手糊成一大團，包括了一個一千五百多行的函數。還需要重構。

所以雖然程式碼是公開的，但不太適合人類閱讀，別人如果想幫我加或改都不太容易。
</aside>
---
<!-- .slide: data-transition="none"-->

## 後續目標

  + <!-- .element: class="fragment" -->單 type array 加速
    <br>預期這個會比現在的 scalar type 加速更多
  + <!-- .element: class="fragment" -->POD hash 加速
  + <!-- .element: class="fragment" -->可加速的 type 支援 reference
  + <!-- .element: class="fragment" -->參數與 return 值有 type 的函數<br>
    這樣遞迴可以變快
  + <!-- .element: class="fragment" -->粗略 track value 的範圍<br>
    做這個才能判斷出長度不會增加的陣列
  + <!-- .element: class="fragment" -->固定長度的 array 更加速

<aside class=notes>
除了完成度之外，這邊還有一些可以再進步的空間。

主要就是支援、認出更多可以加速的 type。
@@
第一個是每個元素都同 type 的 array。這個應該會比現在 scalar type 加速更多。
@@
然後是 plain-old-data 用法的 hash 加速。如果 hash key 都是直接放一個字串 literal 的話，它其實就像 C 的 struct，也沒必要用 hash 的方式來取出欄位，可以直接判斷並寫死它是第幾個欄位。
@@
接下來是讓 reference 也可以用在 static type。目前只要用到 reference，就一定要退回最純正的 Perl 變數。
@@
下一個是讓函數可以提供有 type 的版本。這一步是想讓有 type 與 dynamic type 兩個入口並存。

平常其他 Perl 程式是呼叫 dynamic type 的入口，因為 Perl 那邊的參數與回傳值都是 dynamic type；

而特化過的函數互相呼叫時，可以呼叫 static type 的入口，少一個打包 dynamic type 再拆包的動作，而且也不用經過 Perl 的函數處理，直接在 C level 就完成函數呼叫。

這會讓遞迴程式加速不少。
@@
再下一步是想挑戰粗略地 track 值域。精準地追蹤出若且唯若的值域那是電腦不可計算的 halting problem，不過粗估一下，容許推估的值域中有些值可能根本不會出現，那就還是可以做一下。

做這個的好處，就是可以分離出長度不會增加的陣列，才能實作完全不用 boxed 的單 type 陣列。

不然一般陣列都是至少要搭配迴圈使用，只要 index 放了變數，因為我完全不知道變數的值域範圍，所以就看不出哪個陣列是不會變長的。

只是這一個比起來這一步我沒那麼有把握寫得出來。它理論上可以做，實際要做感覺有點難……
@@
最後就是加速固定長度的陣列。
</aside>
---
<!-- .slide: data-transition="none"-->

## 謝謝聆聽

<img src=me.jpg>

  |  | 社交平台 |
  | -: | - |
  | github | https://github.com/cindylinz/ |
  | CPAN | https://metacpan.org/author/CINDY |
  | Hackage | https://hackage.haskell.org/user/CindyLinz |
  | Facebook | https://facebook.com/cindylinz/ |
  | Twitter | https://twitter.com/cindy_linz/ |

<aside class=notes>
感謝大家。
</aside>
