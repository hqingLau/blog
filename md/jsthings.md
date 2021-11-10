定时点击按钮：

```js
// ==UserScript==
// @name         购物车确认
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @match        https://cart.taobao.com/cart.htm*
// @icon         https://www.google.com/s2/favicons?domain=taobao.com
// @grant        none
// ==/UserScript==

(function() {
    // 网速不同，应该是按照到达服务器的时间算的。
    // ping的往返时间大概25ms，这里提起10ms。
    var setTime = '2021/8/27 10:07:59:990';
    var date = new Date(setTime);
    var time1 = date.valueOf();
    var myTimer;
    function timeClick() {
        var timestamp = (new Date()).valueOf();
        if(timestamp>time1)
        {
            document.getElementById('J_Go').click();
            clearInterval(myTimer);
        }
    }
    setTimeout(function(){
        timeClick();
    },8000);
    myTimer = setInterval(function(){
        timeClick();
    },1);
})();
```

订单确认界面直接模拟点击：

```js
// ==UserScript==
// @name         order confirm 淘宝
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @match        https://buy.taobao.com/auction/order/confirm_order.htm*
// @match        https://buy.tmall.com/order/confirm_order.htm*
// @icon         https://www.google.com/s2/favicons?domain=taobao.com
// @grant        none
// ==/UserScript==

(function() {
    'use strict';
    document.getElementsByClassName('go-btn')[0].click();
})();
```

测试看着效果还可以，等待实操。