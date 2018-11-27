# 小程序的兼容性坑：iOS vs 安卓、真机调试 vs 真机运行

这段时间也搞了一下小程序。小程序声称自己只需要针对自己的平台开发一次，然后各个平台都可以跑，跟之前 java 之类的宣称是一样的。java 结果我们都知道，一次编写，到处调试。小程序也差不多。

## 问题

今天遇到一个兼容性问题症状如下：

小程序在开发环境编译运行没有问题，使用 iPhone 预览没有问题，使用 iPhone 真机调试没有问题，最后发布了体验版、正式版，iPhone 上都完全没有问题。然后接到反馈，说安卓下面这个小程序没办法运行。然后掏出弟弟淘汰了万年的 Meizu M9 看看，果然出了问题，页面只有一个按钮，而且所有功能都没有。

一开始觉得是小事一桩，用真机调试一下，看看什么情况不就完了。结果真机调试二维码扫描连上去，小程序居然就正常了。这就坑了。再尝试了几次，结果是手机直接访问预览版本、体验版本、正式版本都不行，但真机调试就没问题。这个“真机”调试根本就跑不出什么真机的情况，调试环境不一致。提交了下腾讯的自动测试，它跑了几个安卓机型，却又都正常。这就让人想不通了。

由于没办法使用调试工具，而看样子小程序里面的 javascript 代码从一开始就没办法正常运行，有点一筹莫展。然后针对各种猜想做了下尝试。

1. 由于使用了 typescript 的模板，是不是这个模板生成的 javascript 小程序不懂？

直接下手修改了生成后的 javascript 代码，删掉了一些 typescript 生成的内容而觉得不是很需要或者可能出错的。结果没变化。

2. 是不是由于使用了 `import` 来引入 `util` 方法使用，而小程序不支持？

将 `util` 方法都移入到 page 的 js 里，运行，结果依旧。

3. 无奈，最终只好用上了删除一半代码 debug 大法

一路删除定位下来，最终确定是因为安卓系统不支持 `Intl.NumberFormat` 对象。我用这个来格式化显示小程序中的金额数值：

```js
let moneyFormatter = new Intl.NumberFormat('zh-CN', {
    style: 'currency',
    currency: 'CNY',
    minimumFractionDigits: 2
})
return moneyFormatter.format(amount)
```

这个对象现在属于标准的一部分，所有稍微新点的浏览器都支持，NodeJs 也支持，iPhone 上也支持，估计使用的是 javaScriptCore。而安卓上面不知道为什么就不支持，也不知道微信用的是什么样的核心。总之这里存在一个不兼容。

而为什么 __“真机”__ 调试不会出问题呢？猜想是因为真机调试运行的还是在编辑器提供的 chromium 环境上，而 chromium 环境是支持这个 api 的。所以这个“真机”可能也就调试一下触摸操作以及屏幕分辨率方面的东西了。

## 解决方法

用如下几种方法可以解决：

1. 使用 `string.toLocaleString` 方法来代替上面的写法：

```js
return amount.toLocaleString('zh-CN', {
    style: 'currency',
    currency: 'CNY',
    minimumFractionDigits: 2
})
```

这样写，在安卓上不会因为没有 `Intl.NumberFormat` 而出错，虽然它也还是不能正确格式化显示，只会按照原来的数值显示。比如金额是 9998，它就显示成 `9998`，而不是我们希望的 `￥9,998.00`，在 iPhone 上则是没问题的。

2. 使用 `Intl` 的 polyfill

参见这里 https://github.com/andyearnshaw/Intl.js/ 

我没有用这个，因为没有类型信息，不是很喜欢。

3. 自己写了一个 `format` 方法。

由于我需要的场景很简单，就是一个金钱的简单格式化，几行代码就搞定了，引入 polyfill 文件尺寸太大，干脆就自己轮一个了（不作太多全面考虑）。

```ts
export function format(amount: number): string {
  let fixedString = amount.toFixed(2)
  let len = fixedString.length
  let result = ''
  for (let i = len - 1; i >= 0; i--) {
    let stepFromTail = len - 1 - i

    let char = fixedString.charAt(i)
    if (stepFromTail <= 5) {
      result = char + result
    }
    else {
      if (stepFromTail % 3 === 0) {
        if (char !== '-') {
          result = ',' + result
        }
      }
      result = char + result
    }
  }
  return '￥' + result
}
```

## 总结

编写小程序一旦用到各种稍微不是非常常用的 api，一定要留意兼容性问题，而且尽量两个平台都要做一下测试，免得被埋了。