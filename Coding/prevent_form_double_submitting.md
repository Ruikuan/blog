# 避免 form 多次提交

因为网络慢或者鼠标单击变双击等原因，有时候可能会多次点击一个 form 里面的提交按钮，从而导致 form 被多次提交，后台产生不希望的重复操作以及数据。

可以较为简单地通过 jQuery 在浏览器端避免多次提交同一个 form。

```js
$('form').submit(function () {
    $(this).submit(function () { return false; });
});
```

在 form 提交事件中再次注册 提交事件 handler，在这个 handler 里面让后面的提交都无效。这个方案比将 form 内的提交按钮都设置成 disabled 的好，因为设置成 disabled 之后，按钮的 value 就不会提交到服务器端了，是个不希望产生的副作用。