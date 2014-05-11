初级，或者资深初级程序员，永远是从人的角度去看代码，结果就会产生“变量提升”（variable hoisting）之类不存在的概念，误人误己。比如国外某程序员写的[“进阶教学”博客][1]中，有这么一段代码：
```javascript
    function foo() {
        return "A";
    }
    
    var oldFoo = foo;
    
    function foo() {
        return oldFoo() + "B";
    }

    console.log(foo()); // "AB"
```
它的实际结果是 RangeError: Maximum call stack size exceeded ，而在此之前，这位程序员才刚刚讲完什么是“variable hoisting”。

本文旨在教会大家进阶级程序员的第一课，即不再从表面的人的角度去看代码，而是从内在的机器的角度去看代码。而从机器的角度出发，Javascript代码的开始相当于一个全局function的调用，而function在调用后并不是马上一行行地运行下去，而是分为两个阶段：初始设定阶段以及正式运行阶段。
### 设定阶段 ###
例1：
```javascript
    console.log( a,b() )
    var a = 1
    function b() { return 2 }
```
首先在执行前，浏览器会先把这些连着的“字符串”（string）分成一个个“字词”（token），期间会自动在有需要的行尾加上分号，然后再把这些字词组织成一个更方便操作的树状结构（AST），最后才开始执行（execution）。

在设定阶段，会建立一个空的当前代码段（function）的变量表，它可以理解为一个变量名字（identifier）及其内存地址的对照表。

然后会找到 function 这个字词，把它后面的那个字词(b)作为这个function的名字，再把 { 和 } 这两个字词之间的所有字词(这里为 return 2) 和一些相关属性（property）生成一个javascript里的Function object，作为这个function的值。并把这个值的内存地址和名字作为一个binding（绑定）存到变量表里。假如名字重复，则覆盖掉之前的值。（在最开始的例子中，后面的foo就覆盖掉了前面的foo，导致foo/oldFoo老在调用它自己，结果超出了call stack的大小）

这之后会找到 var 这个字词，把它后面的那个字词(a)作为这个variable（变量）的名字，这里会先检查这个名字是否已在变量表中，如果在则直接跳过，因此不会覆盖之前的值，如果不在则分配一个内存地址给它，这个内存地址的值为undefined。最后把它们作为又一个binding存到变量表。

到此设定阶段结束，开始调用 console.log( a,b() ) ，得到的结果为 undefined,2 。

例2：
```javascript
    var a = 1
    b(a)
    function b(c) { console.log(c) }
```
一开始的设定阶段和例1一样，然后先是执行 a = 1，把1赋值给a的内存地址，于是a的值从undefined变为了1。
之后调用 b(a) ，这里会暂停 全局代码 的执行，转而开始执行b。

在b的设定阶段，会先把a的值（1）作为c的值，并存到变量表里，然后由于没有function和var这两个字词，设定阶段结束，开始调用 console.log(c) ，得到的结果为 1 。

例3：
```javascript
    var a = 1
    b(2)
    function b(c) { 
        var d = 3
        console.log(d,c,a,this) 
    }
```
首先还是先声名b和a。而且在b的声名阶段，会把一个b的一个内部变量（无法直接在Javascript里访问），名字叫[[scope]]，设为指向它声名时所在的变量表，在这里即全局的变量表，a和b以及它们的内存地址就是被存在这里。

前面提到的function、var还有{ }等符号都属于浏览器认识的关键字词（keyword），当浏览器读到不认识的字词（名字）时，会首先到当前所在的变量表里找，如果没找到，就会去这个[[scope]]，即上一层的变量表里面找，直到[[scope]]为空（null）时，浏览器就会提示 xxx is not defined 。

这里声名完了后开始执行a=1，之后调用b(2)，，这里会先把把Window，也就是global object（全局对象）的内存地址作为this的值。然后把2赋值给c，之后声名d，于是这一层的变量表即只有c和d。

这时b的设定阶段结束，开始执行 d=3 和 console.log(d,c,a,this ) ，得到的结果为 3,2,1,Window{ ... } 。

【总结】
在function声名时，会设置它的[[scope]]，即它的上一层的变量表，以及[[FormalParameters]]，即所有形式参数的名字，最后是FunctionBody，即 { 和 } 之间的树形结构化了的代码（AST），并用它生成一个Function object作为以后的调用对象。

而代码/function的设定阶段为：
1.设置this，不同情况下它的地址也会不同。
a.对一般function来说，它被设置成global/Window。
b.当以objectXX.methodXX() 形式调用时，它被设置成objectXX。
c.当以new XX() 形式时，它被设置成一个新的空object。
d.当以XX.call()或XX.apply()形式时，它被设置成第一个参数。

2.设定一个内存地址，它指向这个function声名时的[[scope]]。

3.设定这一层的变量表。这一步又依次分为a.分配function的内存地址并赋值；b.如果[[FormalParameters]]里的参数名没有重复，则分配它一个内存地址并赋值；c.如果变量名没重复，则分配它一个内存地址但不赋值。

以上这些共同构成了一个术语叫execution context的东西（代码执行上下文），而设定阶段其实就是把它创建起来的过程。

【ES5】
Ecmascript是Javascript的标准规范，以上的讲解主要是基于它的第3版（ES3），而当前的版本为ES5。（第4版是个失败所以直接到了第5版）

ES3中的变量表就是一个对象叫做VariableObject，这种实现更容易理解，但在访问多层以上的变量时速度会越来越慢，后来的浏览器进行了优化，不再使用VariableObject，优化后的速度基本不会受层数影响。于是ES5里把VariableObject改为了一种抽象的标准，名字为lexical environment（词语环境）。

实际这个lexical environment就是由原来指向[[scope]]的内存地址和VariableObject组成，只不过这个内存地址现在叫outer lexical environment（外部词语环境），而VariableObject现在叫environment record（环境记录），而this在ES5标准里也改名叫thisBinding，并与lexical environment共同构成ES5里的execution context。

ES3和ES5的概念之间的差别其实不大，但却都有一个差别很大的名字，而ES5里还添加了一些这样的东西，因为我认为它们对本文的帮助不大却会大大增加理解难度，所以被我去掉了，想要彻底了解的话可以参考：

http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-1-lexical-environments-common-theory
http://ecma262-5.com/ELS5_HTML_with_CorrectionNotes.htm#Section_10.2

### 运行阶段 ###
在function后面 { 和 } 之间的源代码在浏览器转换过后会变成一个 语句列表（StatementList），而它的每一行都是一条语句（Statement），function的运行其实就是对这一条条语句依次evaluate（求值）的过程。

之所以取名叫evaluate，可能是因为计算机实质上只会做两件事：对值进行 转换（运算）和 传输（载入/保存）。

【转换】
转换通常是通过操作符（operator）实现的，最常见的是+和-等，比较特殊的是==和!等，比较少见的是>>和typeof等。 

而因为Javascript源代码是以UTF-16为格式的一长串字母和符号（string），比如"123"其实是3个16位的"字母" 连成的，所以它在运算前会先转成IEEE 754格式的64位浮点数。（而它最小可以转成一个8位的整数）
undefined、true、false、数字（45092，1.345等）、string（"abc", "a bc defg"等）还有“{ ... }”和“[ ... ]”，这种UTF-16格式的值可以直接转换为它真正的值的“词语”被术语称为literal（字面的）。

除了literal以外，还有两种“词语”会在运算前作转换：

【a.变量】
变量，其实就是一个named value（命名了的值），对变量的转换就是把它的名字变成它的值，方法是去查当前变量表，如果查到，则它的值为绑定的内存地址里的那个值；如果没查到，就去查上层变量表（[[scope]]），如果[[scope]]为空（null）则值它is not defined。

最近新出版了一本书花了90多页专门讲解scope和closure，（不禁让我回想起那些整本都在讲C语言指针的书）在这里我只用2段话：

scope即变量表，包括当前的和所有上层的。（scope中文可以翻译为 一眼可以看到的景物，对于function来说，scope就是 一眼可以看到的变量）

closure，宽泛地说，是function这个概念的一种实现，属于变量的function就是closure，即像其它变量一样可传递和返回的function；（在静态语言的实现中，function无法传递和返回，因为它们的function只有在执行时才得到它需要的变量，执行完后这些变量就消除，而Javascript中function在声名时便生成了一个上层变量表（[[scope]]），只要function不消除这个[[scope]]就不会消除）而具体地说，为了与Javascript中的普通function区分，closure特指那些访问了 [[scope]] 中的变量 的function。

【b.property】
当浏览器看到 a.b 的时候，它不会去变量表里查b的值，因为b是a的一个property。property英文意思除了属性外，更主要的意思是财产，或者说附属品，因此在编程里它的意思可以理解为“附属变量”。

要理解property，首先要理解object。如果说property对应的是文件，那么object对应的就是文件夹，所谓的“一切皆object”，就是把所有的变量都放到一个object里。虽然智商正常的人应该都不会说：“硬盘里的一切皆文件夹”。

正确一点的说法是：“一切皆由object组织“。因为object可以理解为一种结构，从图像思维上看，多个object就会形成一个树状的结构，而这种结构可以用来组织任意数量的人或物，比如现实里的物种/图书/商品的分类，军队/公司/政府的编制等等。

计算机里的数据假如不这么组织，那么一个纯粹的string/array/function就只有它自己，没有property，这样虽然占用的空间少，但像array.length或array.foreach(...)这样方便的操作就无法实现，更不用说 a.b.c.d 这样的链式调用了。

object，这里暂称它为“变量夹”，特殊的地方在于它里面的变量可以“继承”，使得在子变量夹里可以获取到父变量夹里的变量，但通常一个父变量夹里既有需要被继承的变量，也有不需要被继承的，因此Javascript可以在父变量夹里建一个叫prototype的变量夹，用来存放那些需要被继承的变量。之后子变量夹里会自动建一个叫[[proto]]的内部变量，它的值便是它父变量夹的prototype。（的内存地址）

任何object的prototype都可以赋值或更改，但[[proto]]是内置的无法直接更改，它取决于object的创建方式：
```javascript
    var o = { ... }
```
o.[[proto]]指向Object.prototype，而Object.prototype默认是一个空的object（{}），所以没继承任何变量。（这个Object是Javascript里一个全局object变量的名字，所有Javascript里的object都默认以它为原型）
```javascript
    var o = new Xxx( ... )
```
o.[[proto]]指向Xxx.prototype。
```javascript
    var o =  Object.create( Xxx, { ... } )
```
o.[[proto]]指向Xxx，Xxx可以是null也可以是任意object。

所谓的继承，就是当浏览器在当前变量夹找不到某个property时，会自动去它的[[proto]]里面找，依此类推，直到[[proto]]为null为止（property与变量的查找原理一模一样，这就是为什么ES3里会用VariableObject来表示变量表）

比如浏览器在evaluate a.b 时，会先去变量表找a的内存地址，而它值就是一个变量夹，然后再到这个变量夹里找b。而如果是 a[b] 这种形式，则是会先转换b的值（evaluate），这个值如果不是字符串（string）会强制转换为字符串，然后再到a变量夹里找这个字符串对应的变量。

【传递】
计算机语言中最误导人的运算符无疑是 =，它其实不是在表达一种相等的关系，而是把右边求得的值传递到左边求得的 内存地址 上。我想一个更恰当的符号应该是 ←，比如a ← b，a ← 3+5。

而值传递在Javascript中具体还分为两种，直接和间接传递。直接传递通常用于数字或可以用数字表示的值，比如true/false和内存地址（假如CPU是32位则内存地址就是一个32位的2进制数，64位CPU则是64位，依此类推），而比较大的、无法用一个数表示的值，则不会被直接传递，而是只传递这个值所在的内存地址。比如
```javascript
    a = { a:1,b:2,c:3 }
```
这里浏览器在把 { a:1,b:2,c:3 } 转换成对应的值后，会返回这个值所在的内存地址，亦即一个数字，然后把这个数字传递给a在变量表里对应的内存地址。

内存地址之所以是个数字，是因为某种程度上说内存里的每一个字节（byte）都有一个编号，一个1GB（giga byte）的内存就有10亿以上（2的20次方）个编号，而在硬件底层有一个类似于 getMem(编号，大小) 的机器指令，假如编号是123，大小是20，那么这个指令就会把编号123到142里的20个字节取出来。（一台电脑主要有3块内存：CPU内存，显卡内存，声卡内存；想运行软件就要往CPU内存传递数据，想播放声音就要往声卡内存传递数据，想显示图像就要往显卡内存传递数据）

Javascript中除了 =，还有一些其它的值传递，它们理论上说都应该改为用←来表示：
<table>
    <tr>
        <td>{ a: 1,b: 'abc' }</td><td>{ a← 1,b← 'abc' } </td>
    </tr>
    <tr>
    <td>a(1,'abc')</td><td>a← 1,'abc'</td></tr>
    <tr>
    <td>return 1</td><td>1 →</td></tr>
</table>
CPU里有一个寄存器（功能与内存一样，但更快也更贵，只用来存放最经常转换/传递的值），它的值是当前正在执行的机器指令的内存地址，当浏览器往这里传递一个 代表内存地址的数字 时，程序就会跳转到那个地址开始继续执行里面的指令，所以if、while、switch等语句实质上也是在传递值。

具体Javascript里共有多少种操作符和语句，可以参考 [Expressions][2] 和 [Statements][3]。

【理论与实践】
这篇文章里总结了我认为Javascript中最重要的理论知识，应该说理解之后除了EMCA标准外就不用再看理论相关的书或文章了，这时应把时间花在动手实践上，去熟悉和掌握 原生的DOM API、第3方的库/框架/插件、代码的组织与测试等等。

----------


  [1]: http://blog.buymeasoda.com/advanced-javascript-fundamentals/
  [2]: http://es5.github.io/#x11
  [3]: http://es5.github.io/#x12