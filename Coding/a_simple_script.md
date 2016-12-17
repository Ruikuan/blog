# 一个云盘批量文件下载辅助小脚本

#### 生成批量列表  

```javascript
// ==UserScript==
// @name         云盘文件列表
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  不可细说!
// @author       fdk
// @match        http://pan.baidu.com/disk/home*
// @grant        none
// ==/UserScript==

(function() {
    //{access_token} 替换成自己的 token
    let prefix = 'https://www.baidupcs.com/rest/2.0/pcs/stream?method=download&access_token={access_token}&path=';
    let batchText = '';
    let parentPath = $('li[node-type="historylistmanager-history-list"]').children().last().attr('title').replace('全部文件','');
    $('.filename').each(function() {
        let t = $(this);
        let c = t.parent().parent().prev().attr('class');
        if(c.indexOf('dir') < 0) {
            let fullPath = prefix + encodeURIComponent(parentPath + '/' + t.attr('title'));
            batchText = batchText + fullPath  + '\n';
        }
    });
    let textarea = document.createElement('textarea');
    textarea.style.width = window.innerWidth + 'px';
    textarea.style.height = window.innerHeight + 'px';
    textarea.className = 'copyhere';
    textarea.innerHTML = batchText;
    $('.module-list').children().first().before(textarea);
})();
```
  
设置为 context-menu 执行脚本，在云盘页面左边或上边右键点击可执行。  

#### 恢复脚本  

```javascript
// ==UserScript==
// @name         去掉复制遮罩
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  hoho!
// @author       fdk
// @match        http://pan.baidu.com/disk/home*
// @grant        none
// ==/UserScript==

(function() {
    $('.copyhere').remove();
})();
```

同样设置成 context-menu 脚本
