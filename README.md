在开发过程中程序员总会碰到产品经理提出的各种稀奇古怪的需求，尽管有些需求很奇葩，但不得不说有些须有还是能指引我们不断的学习与进步，最近在工作中就碰到这种问题。需求是要求在各主流浏览器上使用自定义的滚动条样式，并且达到完美兼容，此篇博客记录自己的分析过程。此篇博客的源码可访问[slimscroll](https://github.com/Jesse121/slimscroll)

<!-- more -->

为了能使用自定义的滚动条样式并且能在各主流浏览器上兼容，首先想到的就是css定制滚动条的样式。于是在网上搜索了下发现确实有这样的css样式存在：
```css
ul::-webkit-scrollbar/*滚动条整体部分，其中的属性有width,height,background,border（就和一个块级元素一样）等。*/
ul::-webkit-scrollbar-button /*滚动条两端的按钮。可以用display:none让其不显示，也可以添加背景图片，颜色改变显示效果。*/
ul::-webkit-scrollbar-track /* 外层轨道。可以用display:none让其不显示，也可以添加背景图片，颜色改变显示效果。*/
ul::-webkit-scrollbar-track-piece /*内层轨道，滚动条中间部分（除去）。*/
ul::-webkit-scrollbar-thumb  /*滚动条里面可以拖动的那部分*/
ul::-webkit-scrollbar-corner  /* 边角*/
ul::-webkit-resizer /*定义右下角拖动块的样式*/
```
这里只是初略的介绍下，具体的使用方法可参考这篇博客[自定义浏览器滚动条的样式，打造属于你的滚动条风格](https://www.lyblog.net/detail/314.html)  
这种方法确实可以定制滚动条样式，但是仅仅是在webkit内核浏览器有效，不能达到完美兼容。上面的博客中推荐一种jquery插件来解决这个问题，但是介绍的并不全面，也不好使用。于是我在网上搜索了下发现类似的jquery插件有很多，为了更贴近项目需要我选择了jquery.slimscroll.js. 可是在实际使用的过程中发现该插件不支持滚动的内容翻页，一旦翻页追加新内容，因为内容高度的变化导致存在滚动内容跳跃的情况。

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
通过上面的分析我们知道由于在翻页追加新内容后未获取到最新的滚动条的top值和最新的e.pageY值，针对这个问题我在源码中添加了以下一些代码：

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
以上代码解决翻页追加新内容后，拖动滚动条存在的跳跃问题
```js
function _onWheel(e) {
    //... 
    var e = e || window.event;
    
    // 以下是我添加的内容
    //取消拖动
    $(document).unbind('mousemove.slimscroll');
    
    //...
}
```
以上这段代码是为了解决翻页滚动后的点击事件被覆盖问题  

这里为什么用e.buttons而不用e.button?  
MouseEvent.buttons 可指示任意鼠标事件中鼠标的按键情况  
MouseEvent.button 只能够表明在事件中通过按下或者放开一个按键，或者是多个按键同时按下时，哪一个按键被按下。因此，它对判断mouseenter, mouseleave, mouseover, mouseout or mousemove这些事件并不可靠。

**修改源码之后的效果请猛击这里**[demo2](https://jesse121.github.io/slimscroll/demo2.html) 

这里修改的代码仅仅是我一人之见，如果您有更好的方法，欢迎在issue中提出

##### 参考文献：
> [MouseEvent.button](https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/button)  
> [MouseEvent.buttons](https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/buttons)
