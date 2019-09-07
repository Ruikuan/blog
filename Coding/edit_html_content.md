# 直接编辑 html 内容

通常我们编辑网页内容，通常都是使用 html 提供的通用 `form control` 来输入。譬如 `input` `textarea` `select` 等。

不过也存在大量的富文本编辑、代码编辑或者表格编辑的需求。它们是直接在 `div` 等元素中进行输入和编辑的。要实现这点，可以使用下面的两种方法。

### contenteditable

可以在 `htmlElement` 中加入 [`contenteditable`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/contenteditable) 属性，譬如 `<div contenteditable="true"> ... </div>` 这样就可以直接编辑 `div` 中的内容。文档中有 `contenteditable` 的元素后，还可以执行 [`Document.execCommand()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand) 命令，对文档进行操纵。

### 在要编辑的地方插入一个无存在感的 textarea

譬如要编辑 `div` 中的某一段内容，可以获取到那个位置的定位，插入一个无 `border` 的绝对定位的 `textarea`，这样输入的内容就直接输入到了 `textarea` 中，再根据输入的内容进行处理。表格直接编辑也可以利用同样的方法。


这些都是很基本的 html 编辑方法，随便记录一下。