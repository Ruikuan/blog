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

再发一个int类型的指令，和input结合使用
```javascript
Vue.directive('int', {
    twoWay: true,
    bind: function () {
        this.handler = function () {
            let val = this.el.value;
            if (val === '') {
                val = '0';
                this.el.value = val;
            }
            let intVal = parseInt(val, 10);
            if (!isNaN(intVal)) {
                this.set(intVal);
            }

            let currentValue = this._watcher.get();
            this.el.value = currentValue;

        }.bind(this)
        this.el.addEventListener('input', this.handler)
    },
    update: function (val) {
        this.el.value = val;
    },
    unbind: function () {
        this.el.removeEventListener('input', this.handler)
    }
});
```
使用方式
```html
<input type="text" class="form-control" v-int="value" />
```