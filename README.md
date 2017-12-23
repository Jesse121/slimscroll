---
title: 针对模拟滚动条插件(jQuery.slimscroll.js)的修改 
date: 2017-12-23 15:19:24
categories:
- jQuery
tags:
- jQuery
---

为了能使用自定义的滚动条样式并且能在各主流浏览器上兼容，我在网上搜索了下发现类似的jquery插件有很多，为了更贴近项目需要我选择了jquery.slimscroll.js. 可是在实际使用的过程中发现该插件不支持滚动的内容翻页，一旦翻页追加新内容，因为内容高度的变化导致存在跳跃的情况。无奈之下采用css样式自定义滚动条，但这种解决办法只有webkit内核浏览器有效。最近抽出空余时间一直在尝试修改slimscroll插件，此篇博客记录自己的分析过程。

<!-- more -->

### 重现问题
看实际效果请猛击这里[demo1](https://jesse121.github.io/slimscroll/demo1.html)  
从该插件的github上下载源码，并引入到页面中，为了看到效果请打开调试面板，我设置了在内容滚动时输出其scrollTop值，根据输出的结果我们可以很明显的看到在追加新内容后，内容会有跳跃的情况
```
scrollTop 49
scrollTop 51
scrollTop 105   //内容向上跳跃了
scrollTop 108
```

### 打开源码分析原因
源码中的以下这段是在拖动滚动条时要触发的函数
```js
// make it draggable and no longer dependent on the jqueryUI
if (o.railDraggable){ //默认设置为true
  bar.bind("mousedown", function(e) {
    var $doc = $(document);
    isDragg = true;
    t = parseFloat(bar.css('top'));
    pageY = e.pageY;

    $doc.bind("mousemove.slimscroll", function(e){
      currTop = t + e.pageY - pageY;
      bar.css('top', currTop);
      scrollContent(0, bar.position().top, false);// scroll content
    });

    $doc.bind("mouseup.slimscroll", function(e) {
      isDragg = false;hideBar();
      $doc.unbind('.slimscroll');
    });
    return false;
  }).bind("selectstart.slimscroll", function(e){
    e.stopPropagation();
    e.preventDefault();
    return false;
  });
}
```
翻页追加新内容的时候，我们需要重新初始化该插件，初始化的过程中会重新计算滚动条的top值，下拉翻页时由于鼠标一直是按下的状态，在追加新内容后t值和pageY值记录的一直是按下状态时刻(未追加新内容时刻)的。所以当追加完新内容后，再触发mousemove事件时，currTop又会被计算得到翻页之前的值(滚动条的top值也是之前的值了)，而此时的内容高度已经变化了，所以内容会跳跃。(不知道有没有解释清楚啊)

### 针对问题的解决之道
通过上面的分析我们知道由于在翻页追加新内容后未获取到最新的滚动条的top值和最新的e.pageY值，针对这个问题我在源码中添加了一些代码：
```js
// scroll content by the given offset
scrollContent(offset, false, true);
// 以下是我添加的内容
//追加新内容后强制解绑之前的鼠标事件，不使用翻页之前记录的值
$(document).unbind('mousemove.slimscroll');
//解绑后为了能继续下拉滚动条，需重新绑定鼠标事件
//记录最新的滚动条的top值和鼠标位置
var pageY, t = parseFloat(bar.css('top'));
$(document).bind("mousemove.slimscroll", function(e) {
    //这里判断鼠标状态是为了区分滚动事件和拖动事件
    //当鼠标左键是按下状态时，
    if (e.buttons == 1) {
        pageY = pageY || e.pageY
        currTop = t + e.pageY - pageY;
        bar.css('top', currTop);
        scrollContent(0, currTop, false); // scroll content
    }
    return;
})
```
这里为什么用e.buttons而不用e.button?  
MouseEvent.buttons 可指示任意鼠标事件中鼠标的按键情况  
MouseEvent.button 只能够表明在事件中通过按下或者放开一个按键，或者是多个按键同时按下时，哪一个按键被按下。因此，它对判断mouseenter, mouseleave, mouseover, mouseout or mousemove这些事件并不可靠。

修改源码之后的效果请猛击这里[demo2](https://jesse121.github.io/slimscroll/demo1.html) 

##### 参考文献：
> [MouseEvent.button](https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/button)  
> [MouseEvent.buttons](https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/buttons)





