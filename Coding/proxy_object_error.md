# 使用 Proxy Object 的灵异现象和弱智原因

在使用 Vue 开发项目的过程中，一直伴随着一个诡异的问题。应用能在 Edge 上顺利运行，一直没有什么问题，但当跑在 Chorme 上时，开始一段时间可以正常运行，但操作两三次之后就会出错，界面所有操作都失去了程序响应。
说是失去程序响应，是因为界面对各种输入还是响应的，只是输入完全不能改变程序的状态，比如 input 里面输入什么内容，是没办法更改 Vue 里面的 data 的，正常情况下它们互相绑定，数据会随着输入更改。

查看 console，Chorme 出来个奇怪的提示:
```
Error：'set' on proxy: trap returned falsish for property.
```
查了一下 google，不能其解。那个莫名其妙的 falsish 也让我迷惑了，这是啥玩意？是 google 搞出来的错别字吗？ 

为了跟踪对象的更改，以设置某个状态位供自己使用，我的确对 object 进行了一层 proxy 包装，更改了它的 set 行为。一直以来在 Edge 下测试，调试，都完全没有问题，所以我一直没觉得自己的 proxy 有什么问题。
而 Vue 貌似也是用 proxy 来跟踪数据更改的，会不会是它 proxy 了我 proxy 的 object， 所以出了问题？ 跟踪的数据不能进行自己的 proxy？ 用 Vue + 上面那句 Error 搜索，也搜不出任何有用的信息。

是不是只有在 Chorme 上才那么脆弱？怀着这样的念头，我打开了长久闲置的 Firefox。
Firefox的表现跟 Chorme 一致，前几次操作都没有问题，后面就变僵尸了。不过它的 console 输出比较靠谱： 
```
Error：'set' on proxy: trap returned false for property.
``` 
不像 Chorme 那样搞个错别字。

根据多数干掉少数原则，看来有问题的是 Edge，还有我的程序。这次我细细跟搜索结果看了下 proxy 相关的内容，关键是 [MDN 的 handler.set() 说明](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/set) 这个页面，里面有明确说明：
>The set method should return a boolean value. Return true to indicate that assignment succeeded. If the set method returns false, and the assignment happened in strict-mode code, a TypeError will be thrown.  

原来 set 无论如何都是要返回一个 boolean 的，而我的 proxy 知识是在 [Learn ES2015 · Babel](https://babeljs.io/docs/learn-es2015/) 里面快餐式速成的，里面根本没提到返回值这单事，所以我直接就在 set 里面搞完自己的事情就 OK 了，什么返回值都没有给。所以 proxy 的行为算是未定义行为，极可能出问题。诡异的是浏览器并不会一开始就报错，前几次操作成功，后面就出错。

找到原因，问题就容易解决了，我直接在自己的 set 方法里面最后一句加上 return true， 然后什么问题都解决了，三个浏览器都笑了。

综合这段时间的调试，可以看出， Edge 是限制最宽松的，无论是 cors 的凭据控制，还是对 proxy 行为的默许，它都是很随便的就让人通过了。 
Chorme 就严格得多，不过它这种随机性出错也让人抓狂，还有那个错别字是什么鬼？

以后快餐式看完之后，还得找靠谱的地方好好补补课啊。