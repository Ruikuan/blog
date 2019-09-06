# 设置网页打印样式

虽然网页主要是用来在屏幕设备上面查看，但有时候还是会有打印的需求，譬如一些票据、二维码等，需要打印出来使用。下面主要是列举一下打印涉及到的东西。

### 单位

在屏幕上面查看的时候，通常使用 `px` 作为单位。`px` 其实是一个虚拟的单位，粗略来说是表示屏幕能显示一条锐利的细线的宽度，它的显示跟屏幕的参数有很大的关系。同样 `px` 的大小，在手机和电脑屏幕上面显示就不一样，不同的 `ppi` 也不一样。

而在打印中，我们关心的是大小在打印纸上面的表示，距离来说，A4 纸大小中，每个元素在里面占据多大的位置。这种时候我们一般使用 `cm` 或者 `in` 就比较简单。直接去纸上用尺子量一下就知道应该怎么设置边距、大小等数值了。打印会遵循这些物理大小。`pt` 也是一个不错的单位，它也是一个物理单位，`1 in = 72 pt`，可以用来比较精确地微调大小。

### @page

可以用 `@page` 来设置要打印的页面的情况，边距什么的。具体参考 [@page - CSS: Cascading Style Sheets | MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@page)。

### page-break

可以使用 `break-after` `break-before` 等来控制打印的分页。老点的浏览器或者 firefox 好像不支持最新的 `break-after` 等，但支持它们的前辈 `page-break-after` `page-break-before`。可以自行查询使用。


其他的用 `media` 来区分打印和显示设备的，就不说了。

> [Designing For Print With CSS](https://www.smashingmagazine.com/2015/01/designing-for-print-with-css/)  
> [How to Create Printer-friendly Pages with CSS](https://www.sitepoint.com/css-printer-friendly-pages/)