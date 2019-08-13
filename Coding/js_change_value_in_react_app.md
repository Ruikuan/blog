# 使用 native js 改变 React App 中 input 的值

有时候我们需要在 React 上下文之外修改 React App 中的一些输入控件的值，譬如自动填充某些内容。由于 React 的状态渲染流，并不是很容易设置上去。

```js
const changeValue = function(elementid, value) {
         const event = new Event('input', { bubbles: true }); // input 事件对 input 好用
         const changeEvent = new Event('change',{bubbles:true}); // change 事件对 select 好用
         let element = document.getElementById(elementid);
         let lastValue = element.value;
         element.value = value;
         // hack React15
         event.simulated = true;
         // hack React16 内部定义了descriptor拦截value，此处重置状态
         let tracker = element._valueTracker;
         if (tracker) {
             tracker.setValue(lastValue);
         }
         element.dispatchEvent(event);
         element.dispatchEvent(changeEvent);
    };

changeValue('input_name','Zhangsan')
```

> [Trigger simulated input value change for React 16 (after react-dom 15.6.0 updated)?](https://github.com/facebook/react/issues/11488)