# Vue 和 jQuery datepicker 结合使用

最好的结合方式是将 datepicker 封装成vue的自定义指令使用，封装如下：
```javascript
Vue.directive('datepicker', {
    twoWay: true,
    bind: function () {
        $(this.el).datepicker({
            dateFormat: "yy-mm-dd" 
        });
        $(this.el).change(function(){  // change vm value when datepicker pick a date
            this.set($(this.el).val());
        }.bind(this));
    },
    update: function (val) {
        $(this.el).datepicker('setDate', val);
    },
    unbind: function () {
        $(this.el).datepicker('destroy');
        $(this.el).off('change');
    }
});
```
使用方式如下：
```html
<input type="text" class="form-control" v-datepicker="dateValue" />
```
其中dateValue为 data中的值或者v-for里面的元素的值都行。
也可以尝试使用组件的方式实现。